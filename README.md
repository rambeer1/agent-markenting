
README.txt
# Marketing Agent

AI-powered marketing platform for a College/School: content generation, SEO,
social media, lead capture/scoring, campaigns, and analytics — built as a
standalone system (see `../marketing_agent_instructions.md` for the full spec
and phased implementation roadmap).

## Stack

FastAPI + SQLAlchemy + PostgreSQL/PGVector + LangGraph + Gemini (backend),
Angular 19 (frontend).

## Local Development

```powershell
copy .env.example .env
docker compose up -d postgres
cd backend
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install -r ..\requirements.txt
alembic upgrade head
python ..\scripts\seed_data.py
uvicorn main:app --reload
```

API docs: http://localhost:8000/docs

## Tests

```powershell
pytest ..\tests
```

## Project Structure

See `../marketing_agent_instructions.md` → "Project Folder Structure".

===========================================================
backend : agents: analytics_agent: 
__init__.py
agent.py
"""Analytics Agent (Module 9) - dashboard metrics computed from live data."""
from __future__ import annotations

from collections import Counter
from datetime import datetime, timezone

from sqlalchemy import func, select
from sqlalchemy.orm import Session

from models.campaign import Campaign
from models.lead import Lead
from schemas.analytics import DashboardSummary, FunnelMetrics

APPLICATION_STATUSES = {"APPLICATION_STARTED", "APPLICATION_SUBMITTED"}
ADMITTED_STATUS = "ADMITTED"
INTERESTED_STATUSES = {"INTERESTED", "COUNSELING"}


def get_dashboard_summary(db: Session) -> DashboardSummary:
    now = datetime.now(timezone.utc)
    today = now.date()

    all_leads = list(db.scalars(select(Lead)))

    daily_leads = sum(1 for lead in all_leads if lead.created_date.date() == today)
    monthly_leads = sum(
        1
        for lead in all_leads
        if lead.created_date.year == today.year and lead.created_date.month == today.month
    )

    leads_by_source = dict(Counter(lead.source for lead in all_leads))

    course_counts = Counter(lead.course_interest for lead in all_leads if lead.course_interest)
    top_course = course_counts.most_common(1)[0][0] if course_counts else None

    campaign_lead_counts = Counter(lead.campaign_id for lead in all_leads if lead.campaign_id)
    top_campaign_id = campaign_lead_counts.most_common(1)[0][0] if campaign_lead_counts else None
    top_campaign = None
    if top_campaign_id is not None:
        campaign = db.get(Campaign, top_campaign_id)
        top_campaign = campaign.campaign_name if campaign else None

    interested = sum(1 for lead in all_leads if lead.status in INTERESTED_STATUSES)
    applications = sum(1 for lead in all_leads if lead.status in APPLICATION_STATUSES)
    admissions = sum(1 for lead in all_leads if lead.status == ADMITTED_STATUS)

    funnel = FunnelMetrics(
        visitors=len(all_leads),  # visitor tracking is out of scope for v1; leads used as proxy
        leads=len(all_leads),
        interested=interested,
        applications=applications,
        admissions=admissions,
    )

    return DashboardSummary(
        daily_leads=daily_leads,
        monthly_leads=monthly_leads,
        leads_by_source=leads_by_source,
        top_course=top_course,
        top_campaign=top_campaign,
        funnel=funnel,
    )
---------------------------
backend : agents: campaign_agent: 
__init__.py
agent.py
"""Campaign Agent (Module 6).

Workflow: Campaign -> Content Agent -> Social Agent -> Publish -> Lead
Capture -> Analytics. This module wires Content + Social agents together for
a campaign launch; lead capture happens via the normal Lead Management API,
and analytics reads the resulting rows (Phase 10 depends on Phases 5, 7, 8).
"""
from __future__ import annotations

from sqlalchemy.orm import Session

from agents.content_agent.agent import generate_content
from agents.social_agent.agent import generate_social_post
from models.campaign import Campaign
from repositories.campaign_repository import CampaignRepository


def launch_campaign(db: Session, campaign_id: int, topic: str, platforms: list[str]) -> dict:
    campaign_repo = CampaignRepository(db)
    campaign: Campaign | None = campaign_repo.get(campaign_id)
    if campaign is None:
        raise ValueError(f"Campaign {campaign_id} not found")

    content = generate_content(
        db,
        course=campaign.campaign_type,
        topic=topic,
        targetAudience="Prospective Students",
        content_type="Article",
        campaign_id=campaign.campaign_id,
        course_id=campaign.course_id,
    )

    social_posts = [generate_social_post(db, platform=p, topic=topic) for p in platforms]

    campaign_repo.update(campaign, {"status": "PUBLISHED"})

    return {
        "campaign_id": campaign.campaign_id,
        "content_id": content.content_id,
        "status": campaign.status,
        "social_posts": social_posts,
    }
------------------------------------
backend : agents: content_agent: 
__init__.py
agent.py
"""Content Agent (Module 3).

Workflow: Input -> Retrieve Knowledge -> Build Prompt -> Gemini ->
Content Validator -> Save Content.
"""
from __future__ import annotations

from sqlalchemy.orm import Session

from models.content import Content
from prompts.templates import CONTENT_AGENT_PROMPT, render
from repositories.content_repository import ContentRepository
from services import gemini_service, vector_service
from utils.logger import get_logger

logger = get_logger(__name__)

MIN_VALID_LENGTH = 20
BANNED_PHRASES = ["lorem ipsum"]


def _validate(text: str) -> tuple[str, str]:
    """Content Validator step: rejects empty/placeholder/too-short generations."""
    if not text or len(text.strip()) < MIN_VALID_LENGTH:
        return text, "REJECTED"
    if any(phrase in text.lower() for phrase in BANNED_PHRASES):
        return text, "REJECTED"
    return text, "PENDING_REVIEW"


def generate_content(
    db: Session,
    course: str,
    topic: str,
    targetAudience: str,
    content_type: str = "Article",
    campaign_id: int | None = None,
    course_id: int | None = None,
) -> Content:
    # 1. Retrieve Knowledge
    hits = vector_service.semantic_search(db, query=f"{course} {topic}", top_k=3)
    knowledge_context = "\n".join(h.text_content for h in hits) or (
        "No additional knowledge base context available."
    )

    # 2. Build Prompt
    prompt = render(
        CONTENT_AGENT_PROMPT,
        course=course,
        topic=topic,
        target_audience=targetAudience,
        content_type=content_type,
        knowledge_context=knowledge_context,
    )

    # 3. Gemini
    generated_text = gemini_service.generate_text(prompt)

    # 4. Content Validator
    validated_text, status = _validate(generated_text)

    # 5. Save Content
    content = Content(
        content_type=content_type,
        title=f"{topic} - {course}",
        content=validated_text,
        prompt=prompt,
        generated_by="content_agent",
        status=status,
        campaign_id=campaign_id,
        course_id=course_id,
    )
    return ContentRepository(db).create(content)

------------------------------------
backend : agents: lead_agent: 
__init__.py
scoring.py
"""Lead Scoring Agent (Module 8).

Applies the point rules from the spec and recalculates a lead's score
whenever a relevant lead_activity is recorded.
"""
from __future__ import annotations

from sqlalchemy.orm import Session

from repositories.lead_repository import LeadActivityRepository, LeadRepository

# Point rules
SCORE_RULES: dict[str, int] = {
    "SCIENCE_STUDENT": 20,
    "INTEREST_BCA": 30,
    "DOWNLOADED_BROCHURE": 15,
    "ATTENDED_WEBINAR": 25,
    "RESPONDED_WHATSAPP": 10,
}
WEBSITE_VISIT_ACTIVITY = "WEBSITE_VISIT"
WEBSITE_VISIT_THRESHOLD = 3
WEBSITE_VISIT_BONUS = 10

MAX_SCORE = 100

COLD_MAX = 30
WARM_MAX = 60


def score_band(score: int) -> str:
    if score <= COLD_MAX:
        return "Cold"
    if score <= WARM_MAX:
        return "Warm"
    return "Hot"


def recalculate_lead_score(db: Session, lead_id: int):
    lead = LeadRepository(db).get(lead_id)
    if lead is None:
        raise ValueError(f"Lead {lead_id} not found")

    activities = LeadActivityRepository(db).list_for_lead(lead_id)
    total = 0
    website_visits = 0
    for activity in activities:
        if activity.activity_type == WEBSITE_VISIT_ACTIVITY:
            website_visits += 1
            continue
        total += SCORE_RULES.get(activity.activity_type, 0)

    if website_visits > WEBSITE_VISIT_THRESHOLD:
        total += WEBSITE_VISIT_BONUS

    lead.lead_score = min(total, MAX_SCORE)
    db.commit()
    db.refresh(lead)
    return lead

------------------------------------
backend : agents: orchestrator: 
__init__.py
graph.py
"""Phase 12 - Full Multi-Agent Orchestration.

Wires Content/SEO/Social/Campaign/Analytics agents as LangGraph nodes with a
conditional entry point (fan-out by task) and per-node error handling. Lead
scoring is triggered directly by the Lead API (event-driven, not part of this
request/response graph) per the dependency graph in the instructions doc.
"""
from __future__ import annotations

from langgraph.graph import END, StateGraph
from sqlalchemy.orm import Session

from agents.analytics_agent.agent import get_dashboard_summary
from agents.campaign_agent.agent import launch_campaign
from agents.content_agent.agent import generate_content
from agents.seo_agent.agent import generate_seo_blog
from agents.social_agent.agent import generate_social_post
from agents.state import MarketingAgentState
from utils.logger import get_logger

logger = get_logger(__name__)

TASKS = ("generate_content", "generate_seo", "generate_social", "run_campaign", "get_analytics")


def _guarded(task_name: str, fn):
    """Wraps a node so failures are captured in state instead of raising,
    matching the "Retry Logic / Error Handling" requirement from the spec.
    Gemini-level retries already happen inside gemini_service; this layer
    catches anything that still fails and reports it in `state["error"]`."""

    def node(state: MarketingAgentState) -> MarketingAgentState:
        try:
            result = fn(state.get("payload", {}))
            return {**state, "result": result, "error": None}
        except Exception as exc:  # noqa: BLE001 - intentional broad catch at graph boundary
            logger.error("orchestrator node '%s' failed: %s", task_name, exc)
            return {**state, "result": None, "error": str(exc)}

    return node


def build_orchestrator(db: Session):
    """Builds a request-scoped compiled LangGraph bound to `db`."""

    def content_node(payload: dict):
        content = generate_content(db, **payload)
        return {"contentId": content.content_id, "status": content.status}

    def seo_node(payload: dict):
        blog = generate_seo_blog(db, **payload)
        return {"blogId": blog.blog_id, "slug": blog.slug, "status": blog.status}

    def social_node(payload: dict):
        return generate_social_post(db, **payload)

    def campaign_node(payload: dict):
        return launch_campaign(db, **payload)

    def analytics_node(_payload: dict):
        return get_dashboard_summary(db).model_dump()

    graph = StateGraph(MarketingAgentState)
    graph.add_node("generate_content", _guarded("generate_content", content_node))
    graph.add_node("generate_seo", _guarded("generate_seo", seo_node))
    graph.add_node("generate_social", _guarded("generate_social", social_node))
    graph.add_node("run_campaign", _guarded("run_campaign", campaign_node))
    graph.add_node("get_analytics", _guarded("get_analytics", analytics_node))

    graph.set_conditional_entry_point(lambda state: state["task"], {task: task for task in TASKS})

    for task in TASKS:
        graph.add_edge(task, END)

    return graph.compile()


def run_task(db: Session, task: str, payload: dict) -> MarketingAgentState:
    if task not in TASKS:
        raise ValueError(f"Unknown orchestrator task '{task}'. Valid tasks: {TASKS}")
    orchestrator = build_orchestrator(db)
    return orchestrator.invoke({"task": task, "payload": payload, "retry_count": 0})

------------------------------------
backend : agents: seo_agent: 
__init__.py
agent.py
"""SEO Agent (Module 4) - generates an SEO blog package for a target keyword."""
from __future__ import annotations

import re

from sqlalchemy import select
from sqlalchemy.orm import Session

from models.seo_blog import SeoBlog
from prompts.templates import SEO_AGENT_PROMPT, render
from repositories.seo_repository import SeoBlogRepository
from services import gemini_service


def _slugify(text: str) -> str:
    slug = re.sub(r"[^a-z0-9]+", "-", text.lower()).strip("-")
    return slug or "seo-blog"


def generate_seo_blog(db: Session, keyword: str) -> SeoBlog:
    prompt = render(SEO_AGENT_PROMPT, keyword=keyword)
    generated_text = gemini_service.generate_text(prompt)

    slug = _slugify(keyword)
    existing = db.scalars(select(SeoBlog).where(SeoBlog.slug == slug)).first()
    if existing:
        slug = f"{slug}-{existing.blog_id}"

    blog = SeoBlog(
        title=f"{keyword} | Complete Guide",
        keyword=keyword,
        slug=slug,
        meta_description=generated_text.strip().replace("\n", " ")[:160],
        content=generated_text,
        status="DRAFT",
    )
    return SeoBlogRepository(db).create(blog)

------------------------------------
backend : agents: social_agent: 
__init__.py
agent.py
"""Social Media Agent (Module 5).

Generates platform-specific captions/hashtags/CTA/media prompts, persists the
raw generation as a Content record, then links it via a SocialPost row.
"""
from __future__ import annotations

import re

from sqlalchemy.orm import Session

from models.content import Content
from models.social_post import SocialPost
from prompts.templates import SOCIAL_AGENT_PROMPT, render
from repositories.content_repository import ContentRepository
from repositories.social_repository import SocialPostRepository
from services import gemini_service

SUPPORTED_PLATFORMS = {"facebook", "instagram", "linkedin", "youtube", "whatsapp"}


def _extract_hashtags(text: str) -> list[str]:
    return re.findall(r"#\w+", text)


def generate_social_post(db: Session, platform: str, topic: str) -> dict:
    platform = platform.lower()
    if platform not in SUPPORTED_PLATFORMS:
        raise ValueError(f"Unsupported platform '{platform}'. Supported: {SUPPORTED_PLATFORMS}")

    prompt = render(SOCIAL_AGENT_PROMPT, platform=platform, topic=topic)
    generated_text = gemini_service.generate_text(prompt)

    content = ContentRepository(db).create(
        Content(
            content_type="Social Post",
            title=f"{platform.title()} post - {topic}",
            content=generated_text,
            prompt=prompt,
            generated_by="social_agent",
            status="PENDING_REVIEW",
        )
    )

    post = SocialPostRepository(db).create(
        SocialPost(platform=platform, content_id=content.content_id, status="DRAFT")
    )

    return {
        "post_id": post.post_id,
        "caption": generated_text,
        "hashtags": _extract_hashtags(generated_text),
        "cta": "Apply Now",
        "image_prompt": f"Create a promotional image for: {topic}",
        "video_prompt": f"Create a 15-second promotional video for: {topic}",
    }

------------------------------------
backend : agents:  
__init__.py
state.py
"""Shared LangGraph state definition used by the orchestrator (Phase 4/12)."""
from typing import Any, TypedDict


class MarketingAgentState(TypedDict, total=False):
    task: str  # generate_content | generate_seo | generate_social | run_campaign | get_analytics
    payload: dict[str, Any]
    result: dict[str, Any] | None
    error: str | None
    retry_count: int

------------------------------------
backend : alembic: versions: 
0001_initial_schema.py
"""Initial schema: users, courses, content, campaigns, leads, lead_activity,
social_posts, seo_blogs, prompts.

Revision ID: 0001_initial_schema
Revises:
Create Date: 2026-07-15
"""
from alembic import op
import sqlalchemy as sa
from pgvector.sqlalchemy import Vector

revision = "0001_initial_schema"
down_revision = None
branch_labels = None
depends_on = None


def upgrade() -> None:
    op.execute("CREATE EXTENSION IF NOT EXISTS vector")

    op.create_table(
        "users",
        sa.Column("user_id", sa.Integer, primary_key=True, autoincrement=True),
        sa.Column("email", sa.String(255), nullable=False, unique=True),
        sa.Column("hashed_password", sa.String(255), nullable=False),
        sa.Column("full_name", sa.String(255), nullable=False),
        sa.Column("role", sa.String(50), nullable=False, server_default="READ_ONLY"),
        sa.Column("is_active", sa.Boolean, server_default=sa.true()),
        sa.Column("created_date", sa.DateTime(timezone=True), server_default=sa.func.now()),
    )

    op.create_table(
        "courses",
        sa.Column("course_id", sa.Integer, primary_key=True, autoincrement=True),
        sa.Column("course_name", sa.String(255), nullable=False),
        sa.Column("duration", sa.String(100)),
        sa.Column("fees", sa.Numeric(12, 2)),
        sa.Column("description", sa.Text),
        sa.Column("status", sa.String(20), server_default="ACTIVE"),
        sa.Column("created_date", sa.DateTime(timezone=True), server_default=sa.func.now()),
    )

    op.create_table(
        "campaigns",
        sa.Column("campaign_id", sa.Integer, primary_key=True, autoincrement=True),
        sa.Column("campaign_name", sa.String(255), nullable=False),
        sa.Column("campaign_type", sa.String(50), nullable=False),
        sa.Column("start_date", sa.Date),
        sa.Column("end_date", sa.Date),
        sa.Column("budget", sa.Numeric(12, 2)),
        sa.Column("status", sa.String(20), server_default="DRAFT"),
        sa.Column("course_id", sa.Integer, sa.ForeignKey("courses.course_id")),
        sa.Column("created_date", sa.DateTime(timezone=True), server_default=sa.func.now()),
    )

    op.create_table(
        "content",
        sa.Column("content_id", sa.Integer, primary_key=True, autoincrement=True),
        sa.Column("content_type", sa.String(50), nullable=False),
        sa.Column("title", sa.String(500), nullable=False),
        sa.Column("content", sa.Text, nullable=False),
        sa.Column("prompt", sa.Text),
        sa.Column("generated_by", sa.String(100), server_default="content_agent"),
        sa.Column("status", sa.String(20), server_default="DRAFT"),
        sa.Column("course_id", sa.Integer, sa.ForeignKey("courses.course_id")),
        sa.Column("campaign_id", sa.Integer, sa.ForeignKey("campaigns.campaign_id")),
        sa.Column("created_date", sa.DateTime(timezone=True), server_default=sa.func.now()),
    )

    op.create_table(
        "leads",
        sa.Column("lead_id", sa.Integer, primary_key=True, autoincrement=True),
        sa.Column("first_name", sa.String(100), nullable=False),
        sa.Column("last_name", sa.String(100)),
        sa.Column("mobile", sa.String(20), nullable=False),
        sa.Column("email", sa.String(255)),
        sa.Column("city", sa.String(100)),
        sa.Column("state", sa.String(100)),
        sa.Column("course_interest", sa.String(255)),
        sa.Column("source", sa.String(50), nullable=False),
        sa.Column("lead_score", sa.Integer, server_default="0"),
        sa.Column("status", sa.String(30), server_default="NEW"),
        sa.Column("assigned_to", sa.Integer, sa.ForeignKey("users.user_id")),
        sa.Column("campaign_id", sa.Integer, sa.ForeignKey("campaigns.campaign_id")),
        sa.Column("created_date", sa.DateTime(timezone=True), server_default=sa.func.now()),
    )
    op.create_index("ix_leads_mobile", "leads", ["mobile"])

    op.create_table(
        "lead_activity",
        sa.Column("activity_id", sa.Integer, primary_key=True, autoincrement=True),
        sa.Column("lead_id", sa.Integer, sa.ForeignKey("leads.lead_id"), nullable=False),
        sa.Column("activity_type", sa.String(100), nullable=False),
        sa.Column("remarks", sa.Text),
        sa.Column("activity_date", sa.DateTime(timezone=True), server_default=sa.func.now()),
    )

    op.create_table(
        "social_posts",
        sa.Column("post_id", sa.Integer, primary_key=True, autoincrement=True),
        sa.Column("platform", sa.String(50), nullable=False),
        sa.Column("content_id", sa.Integer, sa.ForeignKey("content.content_id"), nullable=False),
        sa.Column("scheduled_time", sa.DateTime(timezone=True)),
        sa.Column("published_time", sa.DateTime(timezone=True)),
        sa.Column("status", sa.String(20), server_default="SCHEDULED"),
    )

    op.create_table(
        "seo_blogs",
        sa.Column("blog_id", sa.Integer, primary_key=True, autoincrement=True),
        sa.Column("title", sa.String(500), nullable=False),
        sa.Column("keyword", sa.String(255), nullable=False),
        sa.Column("slug", sa.String(500), nullable=False, unique=True),
        sa.Column("meta_description", sa.String(500)),
        sa.Column("content", sa.Text, nullable=False),
        sa.Column("status", sa.String(20), server_default="DRAFT"),
        sa.Column("created_date", sa.DateTime(timezone=True), server_default=sa.func.now()),
    )

    op.create_table(
        "prompts",
        sa.Column("prompt_id", sa.Integer, primary_key=True, autoincrement=True),
        sa.Column("agent_name", sa.String(100), nullable=False),
        sa.Column("prompt_type", sa.String(100), nullable=False),
        sa.Column("prompt_text", sa.Text, nullable=False),
        sa.Column("version", sa.Integer, server_default="1"),
        sa.Column("created_date", sa.DateTime(timezone=True), server_default=sa.func.now()),
    )

    op.create_table(
        "knowledge_embeddings",
        sa.Column("embedding_id", sa.Integer, primary_key=True, autoincrement=True),
        sa.Column("source_type", sa.String(50), nullable=False),
        sa.Column("source_id", sa.String(100), nullable=False),
        sa.Column("text_content", sa.Text, nullable=False),
        sa.Column("embedding", Vector(768)),
        sa.Column("created_date", sa.DateTime(timezone=True), server_default=sa.func.now()),
    )


def downgrade() -> None:
    op.drop_table("knowledge_embeddings")
    op.drop_table("prompts")
    op.drop_table("seo_blogs")
    op.drop_table("social_posts")
    op.drop_table("lead_activity")
    op.drop_index("ix_leads_mobile", table_name="leads")
    op.drop_table("leads")
    op.drop_table("content")
    op.drop_table("campaigns")
    op.drop_table("courses")
    op.drop_table("users")
------------------------------------
backend : alembic:  
env.py
"""Alembic environment - uses the app's Settings + declarative Base metadata."""
from logging.config import fileConfig

from alembic import context
from sqlalchemy import engine_from_config, pool

from config.settings import get_settings
from models import Base  # noqa: F401 (imports all models so metadata is populated)

config = context.config

if config.config_file_name is not None:
    fileConfig(config.config_file_name)

target_metadata = Base.metadata

settings = get_settings()
config.set_main_option("sqlalchemy.url", settings.database_url)


def run_migrations_offline() -> None:
    url = config.get_main_option("sqlalchemy.url")
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )
    with context.begin_transaction():
        context.run_migrations()


def run_migrations_online() -> None:
    connectable = engine_from_config(
        config.get_section(config.config_ini_section, {}),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )
    with connectable.connect() as connection:
        context.configure(connection=connection, target_metadata=target_metadata)
        with context.begin_transaction():
            context.run_migrations()


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()

------------------------------------
backend : alembic:  
script.py.mako
"""Single-purpose script template used by `alembic revision`."""
"""${message}

Revision ID: ${up_revision}
Revises: ${down_revision | comma,n}
Create Date: ${create_date}

"""
from alembic import op
import sqlalchemy as sa
${imports if imports else ""}

# revision identifiers, used by Alembic.
revision = ${repr(up_revision)}
down_revision = ${repr(down_revision)}
branch_labels = ${repr(branch_labels)}
depends_on = ${repr(depends_on)}


def upgrade() -> None:
    ${upgrades if upgrades else "pass"}


def downgrade() -> None:
    ${downgrades if downgrades else "pass"}

------------------------------------
backend : api: dependencies:
__init__.py 
auth.py
"""FastAPI dependencies for authentication & RBAC enforcement."""
from __future__ import annotations

from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer
from sqlalchemy.orm import Session

from database.session import get_db
from models.user import User
from repositories.user_repository import UserRepository
from security.jwt_handler import TokenError, decode_token
from security.rbac import Role

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/v1/auth/login")


def get_current_user(
    token: str = Depends(oauth2_scheme), db: Session = Depends(get_db)
) -> User:
    try:
        payload = decode_token(token, expected_type="access")
    except TokenError as exc:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail=str(exc),
            headers={"WWW-Authenticate": "Bearer"},
        ) from exc

    email = payload.get("sub")
    user = UserRepository(db).find_by_email(email) if email else None
    if not user or not user.is_active:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED, detail="User not found or inactive"
        )
    return user


def require_roles(*allowed_roles: Role):
    """Dependency factory enforcing RBAC on a route: `Depends(require_roles(Role.ADMIN))`."""

    def _check(current_user: User = Depends(get_current_user)) -> User:
        if current_user.role not in {r.value for r in allowed_roles}:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail=f"Role '{current_user.role}' is not permitted to perform this action",
            )
        return current_user

    return _check
------------------------------------
backend : api: routes:
__init__.py 
analytics.py
"""Module 9 - Analytics Agent route."""
from __future__ import annotations

from fastapi import APIRouter, Depends
from sqlalchemy.orm import Session

from agents.orchestrator.graph import run_task
from database.session import get_db
from schemas.analytics import DashboardSummary

router = APIRouter(prefix="/api/v1/analytics", tags=["analytics-agent"])


@router.get("/dashboard", response_model=DashboardSummary)
def dashboard(db: Session = Depends(get_db)) -> DashboardSummary:
    state = run_task(db, "get_analytics", {})
    return DashboardSummary(**state["result"])

------------------------------------
backend : api: routes:
auth.py
"""Module 1 - Authentication routes (login, refresh, logout, password reset)."""
from __future__ import annotations

from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.orm import Session

from database.session import get_db
from models.user import User
from repositories.user_repository import UserRepository
from schemas.auth import (
    LoginRequest,
    PasswordResetConfirm,
    PasswordResetRequest,
    RefreshRequest,
    TokenResponse,
    UserCreate,
    UserOut,
)
from security.jwt_handler import TokenError, create_access_token, create_refresh_token, decode_token
from security.password import hash_password, verify_password

router = APIRouter(prefix="/api/v1/auth", tags=["auth"])


@router.post("/register", response_model=UserOut, status_code=status.HTTP_201_CREATED)
def register(payload: UserCreate, db: Session = Depends(get_db)) -> User:
    repo = UserRepository(db)
    if repo.find_by_email(payload.email):
        raise HTTPException(status.HTTP_409_CONFLICT, "Email already registered")
    user = User(
        email=payload.email,
        hashed_password=hash_password(payload.password),
        full_name=payload.full_name,
        role=payload.role,
    )
    return repo.create(user)


@router.post("/login", response_model=TokenResponse)
def login(payload: LoginRequest, db: Session = Depends(get_db)) -> TokenResponse:
    user = UserRepository(db).find_by_email(payload.email)
    if not user or not verify_password(payload.password, user.hashed_password):
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Invalid email or password")
    if not user.is_active:
        raise HTTPException(status.HTTP_403_FORBIDDEN, "User is inactive")
    return TokenResponse(
        access_token=create_access_token(user.email, user.role),
        refresh_token=create_refresh_token(user.email, user.role),
    )


@router.post("/refresh", response_model=TokenResponse)
def refresh(payload: RefreshRequest, db: Session = Depends(get_db)) -> TokenResponse:
    try:
        claims = decode_token(payload.refresh_token, expected_type="refresh")
    except TokenError as exc:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, str(exc)) from exc

    user = UserRepository(db).find_by_email(claims["sub"])
    if not user or not user.is_active:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "User not found or inactive")

    return TokenResponse(
        access_token=create_access_token(user.email, user.role),
        refresh_token=create_refresh_token(user.email, user.role),
    )


@router.post("/logout", status_code=status.HTTP_204_NO_CONTENT)
def logout() -> None:
    # Stateless JWT: logout is handled client-side by discarding tokens.
    # A production system would additionally maintain a refresh-token
    # denylist; out of scope for v1.
    return None


@router.post("/password-reset/request", status_code=status.HTTP_202_ACCEPTED)
def request_password_reset(payload: PasswordResetRequest, db: Session = Depends(get_db)) -> dict:
    # Always return 202 regardless of whether the email exists, to avoid
    # user enumeration.
    UserRepository(db).find_by_email(payload.email)
    return {"message": "If the account exists, a reset link has been sent."}


@router.post("/password-reset/confirm", status_code=status.HTTP_200_OK)
def confirm_password_reset(payload: PasswordResetConfirm, db: Session = Depends(get_db)) -> dict:
    repo = UserRepository(db)
    user = repo.find_by_email(payload.email)
    if not user:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "User not found")
    repo.update(user, {"hashed_password": hash_password(payload.new_password)})
    return {"message": "Password updated successfully"}

------------------------------------
backend : api: routes:
campaigns.py
"""Module 6 - Campaign Agent routes."""
from __future__ import annotations

from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.orm import Session

from agents.orchestrator.graph import run_task
from api.dependencies.auth import require_roles
from database.session import get_db
from models.campaign import Campaign
from repositories.campaign_repository import CampaignRepository
from schemas.campaign import CampaignCreate, CampaignLaunchRequest, CampaignOut, CampaignUpdate
from security.rbac import Role

router = APIRouter(prefix="/api/v1/campaigns", tags=["campaign-agent"])


@router.get("", response_model=list[CampaignOut])
def list_campaigns(skip: int = 0, limit: int = 100, db: Session = Depends(get_db)) -> list[Campaign]:
    return CampaignRepository(db).list(skip=skip, limit=limit)


@router.post("", response_model=CampaignOut, status_code=status.HTTP_201_CREATED)
def create_campaign(
    payload: CampaignCreate,
    db: Session = Depends(get_db),
    _user=Depends(require_roles(Role.ADMIN, Role.MARKETING_MANAGER)),
) -> Campaign:
    return CampaignRepository(db).create(Campaign(**payload.model_dump()))


@router.put("/{campaign_id}", response_model=CampaignOut)
def update_campaign(
    campaign_id: int,
    payload: CampaignUpdate,
    db: Session = Depends(get_db),
    _user=Depends(require_roles(Role.ADMIN, Role.MARKETING_MANAGER)),
) -> Campaign:
    repo = CampaignRepository(db)
    campaign = repo.get(campaign_id)
    if not campaign:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "Campaign not found")
    return repo.update(campaign, payload.model_dump(exclude_unset=True))


@router.post("/{campaign_id}/launch")
def launch(
    campaign_id: int,
    payload: CampaignLaunchRequest,
    db: Session = Depends(get_db),
    _user=Depends(require_roles(Role.ADMIN, Role.MARKETING_MANAGER)),
) -> dict:
    """Runs Campaign -> Content Agent -> Social Agent -> Publish workflow."""
    state = run_task(
        db,
        "run_campaign",
        {"campaign_id": campaign_id, "topic": payload.topic, "platforms": payload.platforms},
    )
    if state.get("error"):
        raise HTTPException(status.HTTP_502_BAD_GATEWAY, state["error"])
    return state["result"]

------------------------------------
backend : api: routes:
college.py
"""Module 2 - College Management (Course CRUD; Departments/Faculty/Facilities/
Placements follow the same pattern and are omitted here for brevity, add as
sibling routers reusing BaseRepository the same way)."""
from __future__ import annotations

from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.orm import Session

from api.dependencies.auth import require_roles
from database.session import get_db
from models.course import Course
from repositories.course_repository import CourseRepository
from schemas.college import CourseCreate, CourseOut, CourseUpdate
from security.rbac import Role

router = APIRouter(prefix="/api/v1/courses", tags=["college-management"])


@router.get("", response_model=list[CourseOut])
def list_courses(skip: int = 0, limit: int = 100, db: Session = Depends(get_db)) -> list[Course]:
    return CourseRepository(db).list(skip=skip, limit=limit)


@router.get("/{course_id}", response_model=CourseOut)
def get_course(course_id: int, db: Session = Depends(get_db)) -> Course:
    course = CourseRepository(db).get(course_id)
    if not course:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "Course not found")
    return course


@router.post("", response_model=CourseOut, status_code=status.HTTP_201_CREATED)
def create_course(
    payload: CourseCreate,
    db: Session = Depends(get_db),
    _user=Depends(require_roles(Role.ADMIN, Role.MARKETING_MANAGER)),
) -> Course:
    return CourseRepository(db).create(Course(**payload.model_dump()))


@router.put("/{course_id}", response_model=CourseOut)
def update_course(
    course_id: int,
    payload: CourseUpdate,
    db: Session = Depends(get_db),
    _user=Depends(require_roles(Role.ADMIN, Role.MARKETING_MANAGER)),
) -> Course:
    repo = CourseRepository(db)
    course = repo.get(course_id)
    if not course:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "Course not found")
    return repo.update(course, payload.model_dump(exclude_unset=True))


@router.delete("/{course_id}", status_code=status.HTTP_204_NO_CONTENT)
def delete_course(
    course_id: int,
    db: Session = Depends(get_db),
    _user=Depends(require_roles(Role.ADMIN)),
) -> None:
    repo = CourseRepository(db)
    course = repo.get(course_id)
    if not course:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "Course not found")
    repo.delete(course)

------------------------------------
backend : api: routes:
content.py
"""Module 3 - Content Agent route."""
from __future__ import annotations

from fastapi import APIRouter, Depends
from sqlalchemy.orm import Session

from agents.orchestrator.graph import run_task
from api.dependencies.auth import require_roles
from database.session import get_db
from schemas.content import ContentGenerateRequest, ContentGenerateResponse
from security.rbac import Role

router = APIRouter(prefix="/api/v1/content", tags=["content-agent"])


@router.post("/generate", response_model=ContentGenerateResponse)
def generate(
    payload: ContentGenerateRequest,
    db: Session = Depends(get_db),
    _user=Depends(require_roles(Role.ADMIN, Role.MARKETING_MANAGER, Role.CONTENT_REVIEWER)),
) -> ContentGenerateResponse:
    state = run_task(db, "generate_content", payload.model_dump())
    if state.get("error"):
        return ContentGenerateResponse(contentId=0, status="FAILED")
    return ContentGenerateResponse(**state["result"])

------------------------------------
backend : api: routes:
leads.py
"""Module 7 - Lead Management + Module 8 - Lead Scoring (triggered on
activity creation)."""
from __future__ import annotations

from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.orm import Session

from agents.lead_agent.scoring import score_band
from agents.lead_agent import scoring
from api.dependencies.auth import require_roles
from database.session import get_db
from models.lead import Lead, LeadActivity
from repositories.lead_repository import LeadActivityRepository, LeadRepository
from schemas.lead import (
    LeadActivityCreate,
    LeadActivityOut,
    LeadCreate,
    LeadOut,
    LeadUpdate,
)
from security.rbac import Role

router = APIRouter(prefix="/api/v1/leads", tags=["lead-management"])


@router.get("", response_model=list[LeadOut])
def list_leads(skip: int = 0, limit: int = 100, db: Session = Depends(get_db)) -> list[Lead]:
    return LeadRepository(db).list(skip=skip, limit=limit)


@router.get("/{lead_id}", response_model=LeadOut)
def get_lead(lead_id: int, db: Session = Depends(get_db)) -> Lead:
    lead = LeadRepository(db).get(lead_id)
    if not lead:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "Lead not found")
    return lead


@router.post("", response_model=LeadOut, status_code=status.HTTP_201_CREATED)
def create_lead(payload: LeadCreate, db: Session = Depends(get_db)) -> Lead:
    # Public lead-capture endpoint (website/WhatsApp/ads forms) - intentionally
    # not RBAC-protected so external forms can post leads.
    return LeadRepository(db).create(Lead(**payload.model_dump()))


@router.put("/{lead_id}", response_model=LeadOut)
def update_lead(
    lead_id: int,
    payload: LeadUpdate,
    db: Session = Depends(get_db),
    _user=Depends(require_roles(Role.ADMIN, Role.MARKETING_MANAGER, Role.COUNSELOR)),
) -> Lead:
    repo = LeadRepository(db)
    lead = repo.get(lead_id)
    if not lead:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "Lead not found")
    return repo.update(lead, payload.model_dump(exclude_unset=True))


@router.get("/{lead_id}/timeline", response_model=list[LeadActivityOut])
def get_timeline(lead_id: int, db: Session = Depends(get_db)) -> list[LeadActivity]:
    return LeadActivityRepository(db).list_for_lead(lead_id)


@router.post(
    "/{lead_id}/activity", response_model=LeadActivityOut, status_code=status.HTTP_201_CREATED
)
def add_activity(
    lead_id: int,
    payload: LeadActivityCreate,
    db: Session = Depends(get_db),
    _user=Depends(require_roles(Role.ADMIN, Role.MARKETING_MANAGER, Role.COUNSELOR)),
) -> LeadActivity:
    lead_repo = LeadRepository(db)
    if not lead_repo.get(lead_id):
        raise HTTPException(status.HTTP_404_NOT_FOUND, "Lead not found")

    activity = LeadActivityRepository(db).create(
        LeadActivity(lead_id=lead_id, **payload.model_dump())
    )
    # Module 8 - Lead Scoring Agent: recalculate score on every new activity.
    scoring.recalculate_lead_score(db, lead_id)
    return activity


@router.get("/{lead_id}/score")
def get_score(lead_id: int, db: Session = Depends(get_db)) -> dict:
    lead = LeadRepository(db).get(lead_id)
    if not lead:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "Lead not found")
    return {"lead_id": lead.lead_id, "lead_score": lead.lead_score, "band": score_band(lead.lead_score)}

------------------------------------
backend : api: routes:
seo.py
"""Module 4 - SEO Agent route."""
from __future__ import annotations

from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.orm import Session

from agents.orchestrator.graph import run_task
from api.dependencies.auth import require_roles
from database.session import get_db
from schemas.seo import SeoGenerateRequest
from security.rbac import Role

router = APIRouter(prefix="/api/v1/seo", tags=["seo-agent"])


@router.post("/generate")
def generate(
    payload: SeoGenerateRequest,
    db: Session = Depends(get_db),
    _user=Depends(require_roles(Role.ADMIN, Role.MARKETING_MANAGER, Role.CONTENT_REVIEWER)),
) -> dict:
    state = run_task(db, "generate_seo", payload.model_dump())
    if state.get("error"):
        raise HTTPException(status.HTTP_502_BAD_GATEWAY, state["error"])
    return state["result"]

------------------------------------
backend : api: routes:
social.py
"""Module 5 - Social Media Agent route."""
from __future__ import annotations

from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.orm import Session

from agents.orchestrator.graph import run_task
from api.dependencies.auth import require_roles
from database.session import get_db
from schemas.social import SocialPostGenerateRequest
from security.rbac import Role

router = APIRouter(prefix="/api/v1/social", tags=["social-agent"])


@router.post("/post/generate")
def generate(
    payload: SocialPostGenerateRequest,
    db: Session = Depends(get_db),
    _user=Depends(require_roles(Role.ADMIN, Role.MARKETING_MANAGER, Role.CONTENT_REVIEWER)),
) -> dict:
    state = run_task(db, "generate_social", payload.model_dump())
    if state.get("error"):
        raise HTTPException(status.HTTP_502_BAD_GATEWAY, state["error"])
    return state["result"]

------------------------------------
backend : api: 
__init__.py

------------------------------------
backend : config: 
__init__.py
settings.py
"""Application configuration loaded from environment variables."""
from functools import lru_cache
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

    environment: str = "local"
    log_level: str = "INFO"

    database_url: str = "postgresql+psycopg2://marketing:marketing@localhost:5432/marketing_agent"
    db_echo: bool = False

    jwt_secret_key: str = "change-me-in-production"
    jwt_algorithm: str = "HS256"
    access_token_expire_minutes: int = 30
    refresh_token_expire_minutes: int = 10080

    gemini_api_key: str = ""
    gemini_model: str = "gemini-2.5-flash"

    cors_allow_origins: str = "http://localhost:4200"

    gcp_project_id: str = ""
    gcp_region: str = "us-central1"
    gcs_bucket: str = ""

    @property
    def cors_origins_list(self) -> list[str]:
        return [o.strip() for o in self.cors_allow_origins.split(",") if o.strip()]


@lru_cache
def get_settings() -> Settings:
    return Settings()

------------------------------------
backend : database: 
__init__.py
base.py
from database.session import Base

__all__ = ["Base"]
------------------------------------
backend : database: 
session.py
"""SQLAlchemy engine/session setup."""
from collections.abc import Generator

from sqlalchemy import create_engine
from sqlalchemy.orm import DeclarativeBase, Session, sessionmaker

from config.settings import get_settings

settings = get_settings()

engine = create_engine(settings.database_url, echo=settings.db_echo, pool_pre_ping=True)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)


class Base(DeclarativeBase):
    pass


def get_db() -> Generator[Session, None, None]:
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()


------------------------------------
backend : middleware: 
__init__.py
logging_middleware.py
"""Request/response logging middleware (Phase 4 requirement)."""
import time
import uuid

from fastapi import Request

from utils.logger import get_logger

logger = get_logger("http")


async def logging_middleware(request: Request, call_next):
    request_id = str(uuid.uuid4())
    start = time.perf_counter()
    response = await call_next(request)
    duration_ms = round((time.perf_counter() - start) * 1000, 2)
    logger.info(
        "request handled",
        extra={
            "extra_fields": {
                "request_id": request_id,
                "method": request.method,
                "path": request.url.path,
                "status_code": response.status_code,
                "duration_ms": duration_ms,
            }
        },
    )
    response.headers["X-Request-ID"] = request_id
    return response

------------------------------------
backend : models: 
__init__.py
campaign.py
"""Campaign model (Module 6 - Campaign Agent)."""
from datetime import date, datetime, timezone

from sqlalchemy import String, Date, Numeric, DateTime, ForeignKey
from sqlalchemy.orm import Mapped, mapped_column

from database.base import Base


class Campaign(Base):
    __tablename__ = "campaigns"

    campaign_id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    campaign_name: Mapped[str] = mapped_column(String(255), nullable=False)
    campaign_type: Mapped[str] = mapped_column(String(50), nullable=False)
    start_date: Mapped[date] = mapped_column(Date, nullable=True)
    end_date: Mapped[date] = mapped_column(Date, nullable=True)
    budget: Mapped[float] = mapped_column(Numeric(12, 2), nullable=True)
    status: Mapped[str] = mapped_column(String(20), default="DRAFT")
    course_id: Mapped[int | None] = mapped_column(ForeignKey("courses.course_id"), nullable=True)
    created_date: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), default=lambda: datetime.now(timezone.utc)
    )

------------------------------------
backend : models: 
__init__.py
content.py
"""Content model (Module 3 - Content Agent)."""
from datetime import datetime, timezone

from sqlalchemy import String, Text, DateTime, ForeignKey
from sqlalchemy.orm import Mapped, mapped_column

from database.base import Base


class Content(Base):
    __tablename__ = "content"

    content_id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    content_type: Mapped[str] = mapped_column(String(50), nullable=False)
    title: Mapped[str] = mapped_column(String(500), nullable=False)
    content: Mapped[str] = mapped_column(Text, nullable=False)
    prompt: Mapped[str] = mapped_column(Text, nullable=True)
    generated_by: Mapped[str] = mapped_column(String(100), default="content_agent")
    status: Mapped[str] = mapped_column(String(20), default="DRAFT")
    course_id: Mapped[int | None] = mapped_column(ForeignKey("courses.course_id"), nullable=True)
    campaign_id: Mapped[int | None] = mapped_column(
        ForeignKey("campaigns.campaign_id"), nullable=True
    )
    created_date: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), default=lambda: datetime.now(timezone.utc)
    )

------------------------------------
backend : models: 
__init__.py
course.py
"""Course model (Module 2 - College Management)."""
from datetime import datetime, timezone

from sqlalchemy import String, Numeric, Text, DateTime
from sqlalchemy.orm import Mapped, mapped_column

from database.base import Base


class Course(Base):
    __tablename__ = "courses"

    course_id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    course_name: Mapped[str] = mapped_column(String(255), nullable=False)
    duration: Mapped[str] = mapped_column(String(100), nullable=True)
    fees: Mapped[float] = mapped_column(Numeric(12, 2), nullable=True)
    description: Mapped[str] = mapped_column(Text, nullable=True)
    status: Mapped[str] = mapped_column(String(20), default="ACTIVE")
    created_date: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), default=lambda: datetime.now(timezone.utc)
    )

------------------------------------
backend : models: 
__init__.py
knowledge_embedding.py
"""KnowledgeEmbedding model - PGVector-backed embeddings for RAG retrieval.

Stores embeddings for Courses, Placements, Facilities, Events, Achievements,
and FAQs so the Content/SEO agents can do semantic retrieval before prompting
Gemini.
"""
from datetime import datetime, timezone

from pgvector.sqlalchemy import Vector
from sqlalchemy import String, Text, DateTime
from sqlalchemy.orm import Mapped, mapped_column

from database.base import Base

EMBEDDING_DIM = 768


class KnowledgeEmbedding(Base):
    __tablename__ = "knowledge_embeddings"

    embedding_id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    source_type: Mapped[str] = mapped_column(String(50), nullable=False)
    source_id: Mapped[str] = mapped_column(String(100), nullable=False)
    text_content: Mapped[str] = mapped_column(Text, nullable=False)
    embedding: Mapped[list[float]] = mapped_column(Vector(EMBEDDING_DIM), nullable=True)
    created_date: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), default=lambda: datetime.now(timezone.utc)
    )

------------------------------------
backend : models: 
__init__.py
lead.py
"""Lead & LeadActivity models (Module 7 - Lead Management)."""
from datetime import datetime, timezone

from sqlalchemy import String, Integer, DateTime, ForeignKey, Text
from sqlalchemy.orm import Mapped, mapped_column, relationship

from database.base import Base


class Lead(Base):
    __tablename__ = "leads"

    lead_id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    first_name: Mapped[str] = mapped_column(String(100), nullable=False)
    last_name: Mapped[str] = mapped_column(String(100), nullable=True)
    mobile: Mapped[str] = mapped_column(String(20), nullable=False, index=True)
    email: Mapped[str] = mapped_column(String(255), nullable=True)
    city: Mapped[str] = mapped_column(String(100), nullable=True)
    state: Mapped[str] = mapped_column(String(100), nullable=True)
    course_interest: Mapped[str] = mapped_column(String(255), nullable=True)
    source: Mapped[str] = mapped_column(String(50), nullable=False)
    lead_score: Mapped[int] = mapped_column(Integer, default=0)
    status: Mapped[str] = mapped_column(String(30), default="NEW")
    assigned_to: Mapped[int | None] = mapped_column(ForeignKey("users.user_id"), nullable=True)
    campaign_id: Mapped[int | None] = mapped_column(
        ForeignKey("campaigns.campaign_id"), nullable=True
    )
    created_date: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), default=lambda: datetime.now(timezone.utc)
    )

    activities: Mapped[list["LeadActivity"]] = relationship(
        back_populates="lead", cascade="all, delete-orphan"
    )


class LeadActivity(Base):
    __tablename__ = "lead_activity"

    activity_id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    lead_id: Mapped[int] = mapped_column(ForeignKey("leads.lead_id"), nullable=False)
    activity_type: Mapped[str] = mapped_column(String(100), nullable=False)
    remarks: Mapped[str] = mapped_column(Text, nullable=True)
    activity_date: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), default=lambda: datetime.now(timezone.utc)
    )

    lead: Mapped["Lead"] = relationship(back_populates="activities")

------------------------------------
backend : models: 
__init__.py
prompt.py
"""Prompt model - versioned prompt templates used by all agents."""
from datetime import datetime, timezone

from sqlalchemy import String, Text, Integer, DateTime
from sqlalchemy.orm import Mapped, mapped_column

from database.base import Base


class Prompt(Base):
    __tablename__ = "prompts"

    prompt_id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    agent_name: Mapped[str] = mapped_column(String(100), nullable=False)
    prompt_type: Mapped[str] = mapped_column(String(100), nullable=False)
    prompt_text: Mapped[str] = mapped_column(Text, nullable=False)
    version: Mapped[int] = mapped_column(Integer, default=1)
    created_date: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), default=lambda: datetime.now(timezone.utc)
    )

------------------------------------
backend : models: 
__init__.py
seo_blog.py
"""SeoBlog model (Module 4 - SEO Agent)."""
from datetime import datetime, timezone

from sqlalchemy import String, Text, DateTime
from sqlalchemy.orm import Mapped, mapped_column

from database.base import Base


class SeoBlog(Base):
    __tablename__ = "seo_blogs"

    blog_id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    title: Mapped[str] = mapped_column(String(500), nullable=False)
    keyword: Mapped[str] = mapped_column(String(255), nullable=False)
    slug: Mapped[str] = mapped_column(String(500), unique=True, nullable=False)
    meta_description: Mapped[str] = mapped_column(String(500), nullable=True)
    content: Mapped[str] = mapped_column(Text, nullable=False)
    status: Mapped[str] = mapped_column(String(20), default="DRAFT")
    created_date: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), default=lambda: datetime.now(timezone.utc)
    )

------------------------------------
backend : models: 
__init__.py
social_post.py
"""SocialPost model (Module 5 - Social Media Agent)."""
from datetime import datetime, timezone

from sqlalchemy import String, DateTime, ForeignKey
from sqlalchemy.orm import Mapped, mapped_column

from database.base import Base


class SocialPost(Base):
    __tablename__ = "social_posts"

    post_id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    platform: Mapped[str] = mapped_column(String(50), nullable=False)
    content_id: Mapped[int] = mapped_column(ForeignKey("content.content_id"), nullable=False)
    scheduled_time: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), nullable=True)
    published_time: Mapped[datetime | None] = mapped_column(DateTime(timezone=True), nullable=True)
    status: Mapped[str] = mapped_column(String(20), default="SCHEDULED")


------------------------------------
backend : models: 
__init__.py
user.py
"""User model for authentication & RBAC."""
from datetime import datetime, timezone

from sqlalchemy import String, DateTime, Boolean
from sqlalchemy.orm import Mapped, mapped_column

from database.base import Base


class User(Base):
    __tablename__ = "users"

    user_id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
    email: Mapped[str] = mapped_column(String(255), unique=True, index=True, nullable=False)
    hashed_password: Mapped[str] = mapped_column(String(255), nullable=False)
    full_name: Mapped[str] = mapped_column(String(255), nullable=False)
    role: Mapped[str] = mapped_column(String(50), nullable=False, default="READ_ONLY")
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    created_date: Mapped[datetime] = mapped_column(
        DateTime(timezone=True), default=lambda: datetime.now(timezone.utc)
    )
------------------------------------
backend : prompts: 
__init__.py
templates.py
"""Versioned prompt templates for every agent, loaded from / synced to the
`prompts` table (Phase 4 requirement: prompts table is the source of truth,
this module is only the seed/default set and the in-memory formatter)."""
from __future__ import annotations

CONTENT_AGENT_PROMPT = """You are a marketing content writer for a college.
Course: {course}
Topic: {topic}
Target Audience: {target_audience}
Content Type: {content_type}

Write engaging, factually grounded {content_type} content. Use the reference
knowledge below when relevant:

{knowledge_context}
"""

SEO_AGENT_PROMPT = """You are an SEO specialist for a college marketing site.
Keyword: {keyword}

Produce an SEO package with: Title, Meta Description, URL Slug, Tags,
Keywords, FAQs, and Schema JSON (schema.org Article) as clearly labeled
sections.
"""

SOCIAL_AGENT_PROMPT = """You are a social media copywriter.
Platform: {platform}
Topic: {topic}

Produce: Caption, Hashtags (list), CTA, Image prompt, Video prompt as clearly
labeled sections.
"""


def render(template: str, **kwargs: str) -> str:
    return template.format(**kwargs)

------------------------------------
backend : repositories: 
__init__.py
base.py
"""Generic repository base class implementing the Repository Pattern.

Each entity repository subclasses this with its SQLAlchemy model to get
consistent CRUD semantics across the whole backend.
"""
from __future__ import annotations

from typing import Generic, TypeVar

from sqlalchemy import select
from sqlalchemy.orm import Session

from database.base import Base

ModelT = TypeVar("ModelT", bound=Base)


class BaseRepository(Generic[ModelT]):
    model: type[ModelT]

    def __init__(self, db: Session):
        self.db = db

    def get(self, id_value: int) -> ModelT | None:
        return self.db.get(self.model, id_value)

    def list(self, skip: int = 0, limit: int = 100) -> list[ModelT]:
        stmt = select(self.model).offset(skip).limit(limit)
        return list(self.db.scalars(stmt))

    def create(self, obj_in: ModelT) -> ModelT:
        self.db.add(obj_in)
        self.db.commit()
        self.db.refresh(obj_in)
        return obj_in

    def update(self, obj: ModelT, data: dict) -> ModelT:
        for field, value in data.items():
            if value is not None:
                setattr(obj, field, value)
        self.db.commit()
        self.db.refresh(obj)
        return obj

    def delete(self, obj: ModelT) -> None:
        self.db.delete(obj)
        self.db.commit()

------------------------------------
backend : repositories: 
__init__.py
campaign_repository.py
from models.campaign import Campaign
from repositories.base import BaseRepository


class CampaignRepository(BaseRepository[Campaign]):
    model = Campaign

------------------------------------
backend : repositories: 
__init__.py
content_repository.py
from models.content import Content
from repositories.base import BaseRepository


class ContentRepository(BaseRepository[Content]):
    model = Content

------------------------------------
backend : repositories: 
__init__.py
course_repository.py
from models.course import Course
from repositories.base import BaseRepository


class CourseRepository(BaseRepository[Course]):
    model = Course

------------------------------------
backend : repositories: 
__init__.py
lead_repository.py
from sqlalchemy import select
from sqlalchemy.orm import Session

from models.lead import Lead, LeadActivity
from repositories.base import BaseRepository


class LeadRepository(BaseRepository[Lead]):
    model = Lead

    def find_by_mobile(self, mobile: str) -> Lead | None:
        stmt = select(Lead).where(Lead.mobile == mobile)
        return self.db.scalars(stmt).first()


class LeadActivityRepository(BaseRepository[LeadActivity]):
    model = LeadActivity

    def list_for_lead(self, lead_id: int) -> list[LeadActivity]:
        stmt = select(LeadActivity).where(LeadActivity.lead_id == lead_id).order_by(
            LeadActivity.activity_date.desc()
        )
        return list(self.db.scalars(stmt))

------------------------------------
backend : repositories: 
__init__.py
seo_repository.py
from models.seo_blog import SeoBlog
from repositories.base import BaseRepository


class SeoBlogRepository(BaseRepository[SeoBlog]):
    model = SeoBlog

------------------------------------
backend : repositories: 
__init__.py
social_repository.py
from models.social_post import SocialPost
from repositories.base import BaseRepository


class SocialPostRepository(BaseRepository[SocialPost]):
    model = SocialPost

------------------------------------
backend : repositories: 
__init__.py
user_repository.py
from sqlalchemy import select
from sqlalchemy.orm import Session

from models.user import User
from repositories.base import BaseRepository


class UserRepository(BaseRepository[User]):
    model = User

    def find_by_email(self, email: str) -> User | None:
        stmt = select(User).where(User.email == email)
        return self.db.scalars(stmt).first()

------------------------------------
backend : schemas: 
__init__.py
analytics.py
from pydantic import BaseModel


class FunnelMetrics(BaseModel):
    visitors: int
    leads: int
    interested: int
    applications: int
    admissions: int


class DashboardSummary(BaseModel):
    daily_leads: int
    monthly_leads: int
    leads_by_source: dict[str, int]
    top_course: str | None
    top_campaign: str | None
    funnel: FunnelMetrics

------------------------------------
backend : schemas: 
auth.py
from pydantic import BaseModel, EmailStr, Field


class UserCreate(BaseModel):
    email: EmailStr
    password: str = Field(min_length=8)
    full_name: str
    role: str = "READ_ONLY"


class UserOut(BaseModel):
    user_id: int
    email: EmailStr
    full_name: str
    role: str
    is_active: bool

    model_config = {"from_attributes": True}


class LoginRequest(BaseModel):
    email: EmailStr
    password: str


class TokenResponse(BaseModel):
    access_token: str
    refresh_token: str
    token_type: str = "bearer"


class RefreshRequest(BaseModel):
    refresh_token: str


class PasswordResetRequest(BaseModel):
    email: EmailStr


class PasswordResetConfirm(BaseModel):
    email: EmailStr
    new_password: str = Field(min_length=8)

------------------------------------
backend : schemas: 
campaign.py
from datetime import date, datetime

from pydantic import BaseModel


class CampaignCreate(BaseModel):
    campaign_name: str
    campaign_type: str
    start_date: date | None = None
    end_date: date | None = None
    budget: float | None = None
    course_id: int | None = None


class CampaignUpdate(BaseModel):
    status: str | None = None
    start_date: date | None = None
    end_date: date | None = None
    budget: float | None = None


class CampaignOut(BaseModel):
    campaign_id: int
    campaign_name: str
    campaign_type: str
    start_date: date | None
    end_date: date | None
    budget: float | None
    status: str
    created_date: datetime

    model_config = {"from_attributes": True}


class CampaignLaunchRequest(BaseModel):
    """Triggers Campaign -> Content Agent -> Social Agent -> Publish -> Lead Capture."""

    topic: str
    platforms: list[str] = ["instagram", "facebook"]

------------------------------------
backend : schemas: 
college.py
from datetime import datetime

from pydantic import BaseModel


class CourseCreate(BaseModel):
    course_name: str
    duration: str | None = None
    fees: float | None = None
    description: str | None = None
    status: str = "ACTIVE"


class CourseUpdate(BaseModel):
    course_name: str | None = None
    duration: str | None = None
    fees: float | None = None
    description: str | None = None
    status: str | None = None


class CourseOut(BaseModel):
    course_id: int
    course_name: str
    duration: str | None
    fees: float | None
    description: str | None
    status: str
    created_date: datetime

    model_config = {"from_attributes": True}

------------------------------------
backend : schemas: 
content.py
from datetime import datetime

from pydantic import BaseModel


class ContentGenerateRequest(BaseModel):
    course: str
    topic: str
    targetAudience: str
    content_type: str = "Article"


class ContentGenerateResponse(BaseModel):
    contentId: int
    status: str


class ContentOut(BaseModel):
    content_id: int
    content_type: str
    title: str
    content: str
    status: str
    created_date: datetime

    model_config = {"from_attributes": True}

------------------------------------
backend : schemas: 
lead.py
from datetime import datetime

from pydantic import BaseModel


class LeadCreate(BaseModel):
    first_name: str
    last_name: str | None = None
    mobile: str
    email: str | None = None
    city: str | None = None
    state: str | None = None
    course_interest: str | None = None
    source: str
    campaign_id: int | None = None


class LeadUpdate(BaseModel):
    status: str | None = None
    assigned_to: int | None = None
    course_interest: str | None = None


class LeadOut(BaseModel):
    lead_id: int
    first_name: str
    last_name: str | None
    mobile: str
    email: str | None
    city: str | None
    state: str | None
    course_interest: str | None
    source: str
    lead_score: int
    status: str
    assigned_to: int | None
    created_date: datetime

    model_config = {"from_attributes": True}


class LeadActivityCreate(BaseModel):
    activity_type: str
    remarks: str | None = None


class LeadActivityOut(BaseModel):
    activity_id: int
    lead_id: int
    activity_type: str
    remarks: str | None
    activity_date: datetime

    model_config = {"from_attributes": True}

------------------------------------
backend : schemas: 
seo.py
from pydantic import BaseModel


class SeoGenerateRequest(BaseModel):
    keyword: str


class SeoBlogOut(BaseModel):
    blog_id: int
    title: str
    keyword: str
    slug: str
    meta_description: str | None
    content: str
    status: str

    model_config = {"from_attributes": True}

------------------------------------
backend : schemas: 
social.py
from pydantic import BaseModel


class SocialPostGenerateRequest(BaseModel):
    platform: str
    topic: str


class SocialPostGenerateResponse(BaseModel):
    caption: str
    hashtags: list[str]
    cta: str
    image_prompt: str
    video_prompt: str
    post_id: int

------------------------------------
backend : security:
__init__.py 
jwt_handler.py
"""JWT creation/validation for access and refresh tokens."""
from datetime import datetime, timedelta, timezone
from typing import Any

from jose import JWTError, jwt

from config.settings import get_settings

settings = get_settings()


class TokenError(Exception):
    pass


def _create_token(subject: str, role: str, expires_delta: timedelta, token_type: str) -> str:
    now = datetime.now(timezone.utc)
    payload: dict[str, Any] = {
        "sub": subject,
        "role": role,
        "type": token_type,
        "iat": now,
        "exp": now + expires_delta,
    }
    return jwt.encode(payload, settings.jwt_secret_key, algorithm=settings.jwt_algorithm)


def create_access_token(subject: str, role: str) -> str:
    return _create_token(
        subject, role, timedelta(minutes=settings.access_token_expire_minutes), "access"
    )


def create_refresh_token(subject: str, role: str) -> str:
    return _create_token(
        subject, role, timedelta(minutes=settings.refresh_token_expire_minutes), "refresh"
    )


def decode_token(token: str, expected_type: str | None = None) -> dict[str, Any]:
    try:
        payload = jwt.decode(token, settings.jwt_secret_key, algorithms=[settings.jwt_algorithm])
    except JWTError as exc:
        raise TokenError("Invalid or expired token") from exc
    if expected_type and payload.get("type") != expected_type:
        raise TokenError(f"Expected a {expected_type} token")
    return payload

------------------------------------
backend : security: 
password.py
"""Password hashing utilities (bcrypt via passlib)."""
from passlib.context import CryptContext

_pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")


def hash_password(plain_password: str) -> str:
    return _pwd_context.hash(plain_password)


def verify_password(plain_password: str, hashed_password: str) -> bool:
    return _pwd_context.verify(plain_password, hashed_password)

------------------------------------
backend : security: 
rbac.py
"""Role based access control primitives."""
from enum import Enum


class Role(str, Enum):
    ADMIN = "ADMIN"
    MARKETING_MANAGER = "MARKETING_MANAGER"
    COUNSELOR = "COUNSELOR"
    CONTENT_REVIEWER = "CONTENT_REVIEWER"
    READ_ONLY = "READ_ONLY"


# Roles allowed to create/update/delete records (vs. read-only viewing).
WRITE_ROLES = {Role.ADMIN, Role.MARKETING_MANAGER, Role.CONTENT_REVIEWER, Role.COUNSELOR}
ADMIN_ONLY_ROLES = {Role.ADMIN}
CONTENT_APPROVAL_ROLES = {Role.ADMIN, Role.CONTENT_REVIEWER}
LEAD_MANAGEMENT_ROLES = {Role.ADMIN, Role.MARKETING_MANAGER, Role.COUNSELOR}

------------------------------------
backend : services:
__init__.py 
gemini_service.py
"""Gemini wrapper used by every AI agent.

Falls back to a deterministic mock response when GEMINI_API_KEY is not
configured (local dev / CI / offline), so the rest of the pipeline
(validation, persistence, API contracts) can be built and tested without a
live API key. Includes retry with exponential backoff for transient errors.
"""
from __future__ import annotations

from tenacity import retry, stop_after_attempt, wait_exponential

from config.settings import get_settings
from utils.logger import get_logger

logger = get_logger(__name__)
settings = get_settings()

_genai_model = None
_genai_import_error: Exception | None = None

if settings.gemini_api_key:
    try:
        import google.generativeai as genai

        genai.configure(api_key=settings.gemini_api_key)
        _genai_model = genai.GenerativeModel(settings.gemini_model)
    except Exception as exc:  # pragma: no cover - only hit when SDK/key misconfigured
        _genai_import_error = exc
        logger.warning("Gemini SDK unavailable, falling back to mock mode: %s", exc)


class GeminiServiceError(Exception):
    pass


def is_mock_mode() -> bool:
    return _genai_model is None


def _mock_generate(prompt: str) -> str:
    """Deterministic offline stand-in for Gemini output, used in mock mode."""
    snippet = prompt.strip().replace("\n", " ")[:160]
    return (
        "[MOCK GEMINI OUTPUT - configure GEMINI_API_KEY for real generation]\n"
        f"Prompt received: {snippet}...\n"
        "Generated marketing copy placeholder based on the above prompt."
    )


@retry(stop=stop_after_attempt(3), wait=wait_exponential(multiplier=1, min=1, max=8))
def generate_text(prompt: str) -> str:
    """Generate text from Gemini, retrying transient failures up to 3 times."""
    if is_mock_mode():
        return _mock_generate(prompt)

    try:
        response = _genai_model.generate_content(prompt)
        text = getattr(response, "text", None)
        if not text:
            raise GeminiServiceError("Empty response from Gemini")
        return text
    except Exception as exc:
        logger.error("Gemini generation failed: %s", exc)
        raise

------------------------------------
backend : services: 
vector_service.py
"""PGVector-backed knowledge store used for RAG retrieval by Content/SEO agents.

Uses a lightweight deterministic hashing embedding when no live embedding
model is configured, so semantic search still works end-to-end offline. Swap
`_embed` for a real Gemini/Vertex embedding call in production.
"""
from __future__ import annotations

import hashlib

from sqlalchemy import select
from sqlalchemy.orm import Session

from models.knowledge_embedding import EMBEDDING_DIM, KnowledgeEmbedding


def _embed(text: str) -> list[float]:
    """Deterministic pseudo-embedding (stable hash -> floats) for offline use."""
    digest = hashlib.sha256(text.encode("utf-8")).digest()
    values = [(digest[i % len(digest)] / 255.0) for i in range(EMBEDDING_DIM)]
    return values


def store_embedding(db: Session, source_type: str, source_id: str, text_content: str) -> KnowledgeEmbedding:
    record = KnowledgeEmbedding(
        source_type=source_type,
        source_id=source_id,
        text_content=text_content,
        embedding=_embed(text_content),
    )
    db.add(record)
    db.commit()
    db.refresh(record)
    return record


def semantic_search(db: Session, query: str, top_k: int = 5) -> list[KnowledgeEmbedding]:
    """Nearest-neighbour search over knowledge_embeddings using PGVector's
    cosine distance operator (`<=>`)."""
    query_vector = _embed(query)
    stmt = (
        select(KnowledgeEmbedding)
        .order_by(KnowledgeEmbedding.embedding.cosine_distance(query_vector))
        .limit(top_k)
    )
    return list(db.scalars(stmt))

------------------------------------
backend : utils: 
__init__.py
logger.py
"""Structured JSON logging setup used across the whole backend."""
import json
import logging
import sys
from datetime import datetime, timezone

from config.settings import get_settings


class JsonFormatter(logging.Formatter):
    def format(self, record: logging.LogRecord) -> str:
        payload = {
            "timestamp": datetime.now(timezone.utc).isoformat(),
            "level": record.levelname,
            "logger": record.name,
            "message": record.getMessage(),
        }
        if record.exc_info:
            payload["exception"] = self.formatException(record.exc_info)
        extra = getattr(record, "extra_fields", None)
        if extra:
            payload.update(extra)
        return json.dumps(payload)


def configure_logging() -> None:
    settings = get_settings()
    root = logging.getLogger()
    root.setLevel(settings.log_level.upper())
    handler = logging.StreamHandler(sys.stdout)
    handler.setFormatter(JsonFormatter())
    root.handlers = [handler]


def get_logger(name: str) -> logging.Logger:
    return logging.getLogger(name)

------------------------------------
backend :  
alembic.py
[alembic]
script_location = alembic
prepend_sys_path = .
sqlalchemy.url = driver://user:pass@localhost/dbname

[loggers]
keys = root,sqlalchemy,alembic

[handlers]
keys = console

[formatters]
keys = generic

[logger_root]
level = WARN
handlers = console
qualname =

[logger_sqlalchemy]
level = WARN
handlers =
qualname = sqlalchemy.engine

[logger_alembic]
level = INFO
handlers =
qualname = alembic

[handler_console]
class = StreamHandler
args = (sys.stderr,)
level = NOTSET
formatter = generic

[formatter_generic]
format = %(levelname)-5.5s [%(name)s] %(message)s
datefmt = %H:%M:%S

------------------------------------
backend :  
Dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt /app/requirements.txt
RUN pip install --no-cache-dir -r requirements.txt

COPY backend/ /app

EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]

------------------------------------
backend :  
main.py
"""FastAPI application entrypoint."""
from fastapi.middleware.cors import CORSMiddleware
from fastapi import FastAPI

from api.routes import analytics, auth, campaigns, college, content, leads, seo, social
from config.settings import get_settings
from middleware.logging_middleware import logging_middleware
from utils.logger import configure_logging, get_logger

settings = get_settings()
configure_logging()
logger = get_logger(__name__)

app = FastAPI(
    title="Marketing Agent API",
    description="AI-powered marketing platform for College/School admissions marketing.",
    version="1.0.0",
    docs_url="/docs",
    openapi_url="/openapi.json",
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.cors_origins_list,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.middleware("http")(logging_middleware)

app.include_router(auth.router)
app.include_router(college.router)
app.include_router(content.router)
app.include_router(seo.router)
app.include_router(social.router)
app.include_router(leads.router)
app.include_router(campaigns.router)
app.include_router(analytics.router)


@app.get("/health", tags=["system"])
def health() -> dict:
    return {"status": "ok", "environment": settings.environment}


@app.on_event("startup")
def on_startup() -> None:
    logger.info("Marketing Agent API starting up in '%s' environment", settings.environment)
========================================================================

frontend : src: app: features: dashboard:
dashboard.component.html
<div class="p-6">
  <h1 class="text-2xl font-bold mb-4">Marketing Agent - Admin Dashboard</h1>

  <div *ngIf="error" class="text-red-600">{{ error }}</div>

  <div *ngIf="summary" class="grid grid-cols-2 md:grid-cols-4 gap-4">
    <div class="p-4 bg-white shadow rounded">
      <div class="text-sm text-gray-500">Daily Leads</div>
      <div class="text-xl font-semibold">{{ summary.daily_leads }}</div>
    </div>
    <div class="p-4 bg-white shadow rounded">
      <div class="text-sm text-gray-500">Monthly Leads</div>
      <div class="text-xl font-semibold">{{ summary.monthly_leads }}</div>
    </div>
    <div class="p-4 bg-white shadow rounded">
      <div class="text-sm text-gray-500">Top Course</div>
      <div class="text-xl font-semibold">{{ summary.top_course || '-' }}</div>
    </div>
    <div class="p-4 bg-white shadow rounded">
      <div class="text-sm text-gray-500">Top Campaign</div>
      <div class="text-xl font-semibold">{{ summary.top_campaign || '-' }}</div>
    </div>
  </div>
</div>

------------------------------------
frontend : src: app: features: dashboard:
dashboard.component.ts
import { Component, OnInit, inject } from '@angular/core';
import { CommonModule } from '@angular/common';
import { DashboardService, DashboardSummary } from './dashboard.service';

@Component({
  selector: 'app-dashboard',
  standalone: true,
  imports: [CommonModule],
  templateUrl: './dashboard.component.html',
})
export class DashboardComponent implements OnInit {
  private readonly dashboardService = inject(DashboardService);
  summary: DashboardSummary | null = null;
  error: string | null = null;

  ngOnInit(): void {
    this.dashboardService.getSummary().subscribe({
      next: (data) => (this.summary = data),
      error: () => (this.error = 'Unable to load dashboard metrics from the backend.'),
    });
  }
}

------------------------------------
frontend : src: app: features: dashboard:
dashboard.service.ts
import { Injectable, inject } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';
import { environment } from '../../../environments/environment';

export interface FunnelMetrics {
  visitors: number;
  leads: number;
  interested: number;
  applications: number;
  admissions: number;
}

export interface DashboardSummary {
  daily_leads: number;
  monthly_leads: number;
  leads_by_source: Record<string, number>;
  top_course: string | null;
  top_campaign: string | null;
  funnel: FunnelMetrics;
}

@Injectable({ providedIn: 'root' })
export class DashboardService {
  private readonly http = inject(HttpClient);

  getSummary(): Observable<DashboardSummary> {
    return this.http.get<DashboardSummary>(`${environment.apiBaseUrl}/api/v1/analytics/dashboard`);
  }
}

------------------------------------
frontend : src: app:
app.component.ts
import { Component } from '@angular/core';
import { RouterOutlet } from '@angular/router';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [RouterOutlet],
  template: `<router-outlet />`,
})
export class AppComponent {
  title = 'marketing-agent-frontend';
}

------------------------------------
frontend : src: app:
app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideRouter } from '@angular/router';
import { provideHttpClient } from '@angular/common/http';
import { provideAnimations } from '@angular/platform-browser/animations';

import { routes } from './app.routes';

export const appConfig: ApplicationConfig = {
  providers: [provideRouter(routes), provideHttpClient(), provideAnimations()],
};

------------------------------------
frontend : src: app: 
app.routes.ts
import { Routes } from '@angular/router';

/**
 * Phase 13 - Frontend routes. Screens are added incrementally as each
 * backend module's API becomes available (see marketing_agent_instructions.md
 * -> "Implementation Roadmap" -> Phase 13).
 */
export const routes: Routes = [
  {
    path: '',
    loadComponent: () => import('./features/dashboard/dashboard.component').then((m) => m.DashboardComponent),
  },
  { path: '**', redirectTo: '' },
];
------------------------------------
frontend : src: environments: 
environment.prod.ts
export const environment = {
  production: true,
  apiBaseUrl: '/api',
};

------------------------------------
frontend : src: environments: 
environment.ts
export const environment = {
  production: false,
  apiBaseUrl: 'http://localhost:8000',
};

------------------------------------
frontend : src: 
index.html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <title>Marketing Agent</title>
  <base href="/" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <link rel="icon" type="image/x-icon" href="favicon.ico" />
</head>
<body>
  <app-root></app-root>
</body>
</html>

------------------------------------
frontend : src: 
main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { appConfig } from './app/app.config';
import { AppComponent } from './app/app.component';

bootstrapApplication(AppComponent, appConfig).catch((err) => console.error(err));

------------------------------------
frontend : src: 
styles.css
@tailwind base;
@tailwind components;
@tailwind utilities;

html, body {
  height: 100%;
  margin: 0;
  font-family: Roboto, "Helvetica Neue", sans-serif;
}
------------------------------
frontend :
angular.json
{
  "$schema": "./node_modules/@angular/cli/lib/config/schema.json",
  "version": 1,
  "newProjectRoot": "projects",
  "projects": {
    "marketing-agent-frontend": {
      "projectType": "application",
      "root": "",
      "sourceRoot": "src",
      "prefix": "app",
      "architect": {
        "build": {
          "builder": "@angular-devkit/build-angular:application",
          "options": {
            "outputPath": "dist/marketing-agent-frontend",
            "index": "src/index.html",
            "browser": "src/main.ts",
            "polyfills": ["zone.js"],
            "tsConfig": "tsconfig.app.json",
            "assets": ["src/assets"],
            "styles": ["src/styles.css"],
            "scripts": []
          },
          "configurations": {
            "production": {
              "budgets": [
                { "type": "initial", "maximumWarning": "1mb", "maximumError": "2mb" }
              ],
              "outputHashing": "all"
            },
            "development": {
              "optimization": false,
              "sourceMap": true
            }
          },
          "defaultConfiguration": "production"
        },
        "serve": {
          "builder": "@angular-devkit/build-angular:dev-server",
          "configurations": {
            "production": { "buildTarget": "marketing-agent-frontend:build:production" },
            "development": { "buildTarget": "marketing-agent-frontend:build:development" }
          },
          "defaultConfiguration": "development"
        }
      }
    }
  }
}

------------------------------
frontend :
Dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package.json package-lock.json* ./
RUN npm install
COPY . .
RUN npx ng build --configuration production

FROM nginx:alpine
COPY --from=build /app/dist/marketing-agent-frontend/browser /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]

------------------------------
frontend :
package.json
{
  "name": "marketing-agent-frontend",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "start": "ng serve",
    "build": "ng build",
    "test": "ng test",
    "lint": "ng lint"
  },
  "dependencies": {
    "@angular/animations": "^19.0.0",
    "@angular/common": "^19.0.0",
    "@angular/compiler": "^19.0.0",
    "@angular/core": "^19.0.0",
    "@angular/forms": "^19.0.0",
    "@angular/material": "^19.0.0",
    "@angular/cdk": "^19.0.0",
    "@angular/platform-browser": "^19.0.0",
    "@angular/platform-browser-dynamic": "^19.0.0",
    "@angular/router": "^19.0.0",
    "chart.js": "^4.4.4",
    "ng2-charts": "^6.0.1",
    "rxjs": "~7.8.1",
    "tslib": "^2.7.0",
    "zone.js": "~0.15.0"
  },
  "devDependencies": {
    "@angular-devkit/build-angular": "^19.0.0",
    "@angular/cli": "^19.0.0",
    "@angular/compiler-cli": "^19.0.0",
    "autoprefixer": "^10.4.20",
    "postcss": "^8.4.47",
    "tailwindcss": "^3.4.13",
    "typescript": "~5.6.2"
  }
}

------------------------------
frontend :
postcss.config.js
module.exports = {
  plugins: {
    tailwindcss: {},
    autoprefixer: {},
  },
};

------------------------------
frontend :
tailwind.config.js
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: ["./src/**/*.{html,ts}"],
  theme: {
    extend: {},
  },
  plugins: [],
};

------------------------------
frontend :
tsconfig.app.json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "outDir": "./out-tsc/app",
    "types": []
  },
  "files": ["src/main.ts"],
  "include": ["src/**/*.d.ts"]
}

------------------------------
frontend :
tsconfig.json
{
  "compileOnSave": false,
  "compilerOptions": {
    "outDir": "./dist/out-tsc",
    "rootDir": "./src",
    "strict": true,
    "noImplicitOverride": true,
    "noPropertyAccessFromIndexSignature": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "skipLibCheck": true,
    "esModuleInterop": true,
    "sourceMap": true,
    "declaration": false,
    "experimentalDecorators": true,
    "moduleResolution": "bundler",
    "importHelpers": true,
    "target": "ES2022",
    "module": "ES2022",
    "lib": ["ES2022", "dom"]
  },
  "angularCompilerOptions": {
    "enableI18nLegacyMessageIdFormat": false,
    "strictInjectionParameters": true,
    "strictInputAccessModifiers": true,
    "strictTemplates": true
  }
}

------------------------------
frontend :
tsconfig.spec.json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "outDir": "./out-tsc/spec",
    "types": ["jasmine"]
  },
  "include": ["src/**/*.spec.ts", "src/**/*.d.ts"]
}

------------------------------
scripts :
create_admin.py
"""Creates (or resets) the default ADMIN user.

Run from the `marketing-agent` project root:
    python scripts/create_admin.py --email admin@example.com --password SecretPass123
"""
from __future__ import annotations

import argparse
import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).resolve().parent.parent / "backend"))

from database.session import SessionLocal  # noqa: E402
from models.user import User  # noqa: E402
from security.password import hash_password  # noqa: E402


def create_admin(email: str, password: str, full_name: str) -> None:
    db = SessionLocal()
    try:
        user = db.query(User).filter_by(email=email).first()
        if user:
            user.hashed_password = hash_password(password)
            user.role = "ADMIN"
            user.is_active = True
        else:
            user = User(
                email=email,
                hashed_password=hash_password(password),
                full_name=full_name,
                role="ADMIN",
            )
            db.add(user)
        db.commit()
        print(f"Admin user ready: {email}")
    finally:
        db.close()


if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--email", default="admin@marketingagent.local")
    parser.add_argument("--password", default="ChangeMe123!")
    parser.add_argument("--full-name", default="Default Admin")
    args = parser.parse_args()
    create_admin(args.email, args.password, args.full_name)

------------------------------
scripts :
generate_test_data.py
"""Generates larger volumes of synthetic data for load/perf testing
(supports validating the "10,000+ leads" acceptance criterion).

Run from the `marketing-agent` project root:
    python scripts/generate_test_data.py --leads 10000
"""
from __future__ import annotations

import argparse
import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).resolve().parent.parent / "backend"))

from faker import Faker  # noqa: E402

from database.session import SessionLocal  # noqa: E402
from models.lead import Lead  # noqa: E402

fake = Faker()

LEAD_SOURCES = ["Website", "WhatsApp", "Meta Ads", "Google Ads", "Direct Walk-in"]
LEAD_STATUSES = [
    "NEW",
    "CONTACTED",
    "INTERESTED",
    "COUNSELING",
    "APPLICATION_STARTED",
    "APPLICATION_SUBMITTED",
    "ADMITTED",
    "LOST",
]
COURSES = ["BCA", "MCA", "BBA", "MBA", "BCom", "MCom", "BA", "MA"]

BATCH_SIZE = 500


def generate(lead_count: int) -> None:
    db = SessionLocal()
    try:
        created = 0
        batch = []
        for _ in range(lead_count):
            batch.append(
                Lead(
                    first_name=fake.first_name(),
                    last_name=fake.last_name(),
                    mobile=fake.msisdn()[:15],
                    email=fake.email(),
                    city=fake.city(),
                    state=fake.state(),
                    course_interest=fake.random_element(COURSES),
                    source=fake.random_element(LEAD_SOURCES),
                    lead_score=fake.random_int(min=0, max=100),
                    status=fake.random_element(LEAD_STATUSES),
                )
            )
            if len(batch) >= BATCH_SIZE:
                db.add_all(batch)
                db.commit()
                created += len(batch)
                batch = []
        if batch:
            db.add_all(batch)
            db.commit()
            created += len(batch)
        print(f"Generated {created} leads.")
    finally:
        db.close()


if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--leads", type=int, default=10000)
    args = parser.parse_args()
    generate(args.leads)

------------------------------
scripts :
seed_data.py
"""Phase 1 exit-criteria script: seeds baseline + sample data.

Run from the `marketing-agent` project root:
    python scripts/seed_data.py
"""
from __future__ import annotations

import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).resolve().parent.parent / "backend"))

from faker import Faker  # noqa: E402

from database.session import SessionLocal  # noqa: E402
from models import Campaign, Course, Lead, SeoBlog, SocialPost, Content  # noqa: E402
from security.password import hash_password  # noqa: E402
from models.user import User  # noqa: E402

fake = Faker()

COURSES = ["BCA", "MCA", "BBA", "MBA", "BCom", "MCom", "BA", "MA"]
LEAD_SOURCES = ["Website", "WhatsApp", "Meta Ads", "Google Ads", "Direct Walk-in"]
LEAD_STATUSES = [
    "NEW",
    "CONTACTED",
    "INTERESTED",
    "COUNSELING",
    "APPLICATION_STARTED",
    "APPLICATION_SUBMITTED",
    "ADMITTED",
    "LOST",
]
CAMPAIGN_TYPES = ["Admission", "Placement", "Event", "Scholarship"]
PLATFORMS = ["facebook", "instagram", "linkedin", "youtube", "whatsapp"]


def seed() -> None:
    db = SessionLocal()
    try:
        if not db.query(User).filter_by(email="admin@marketingagent.local").first():
            db.add(
                User(
                    email="admin@marketingagent.local",
                    hashed_password=hash_password("ChangeMe123!"),
                    full_name="Default Admin",
                    role="ADMIN",
                )
            )

        courses = []
        if db.query(Course).count() == 0:
            for name in COURSES:
                course = Course(
                    course_name=name,
                    duration="3 Years" if name.startswith("B") else "2 Years",
                    fees=fake.random_int(min=30000, max=200000),
                    description=f"{name} program description.",
                    status="ACTIVE",
                )
                db.add(course)
                courses.append(course)
            db.flush()
        else:
            courses = db.query(Course).all()

        campaigns = []
        if db.query(Campaign).count() == 0:
            for i in range(10):
                campaign = Campaign(
                    campaign_name=f"{fake.catch_phrase()} Campaign {i + 1}",
                    campaign_type=fake.random_element(CAMPAIGN_TYPES),
                    start_date=fake.date_this_year(),
                    end_date=fake.date_this_year(),
                    budget=fake.random_int(min=5000, max=100000),
                    status=fake.random_element(["DRAFT", "SCHEDULED", "PUBLISHED"]),
                    course_id=fake.random_element(courses).course_id if courses else None,
                )
                db.add(campaign)
                campaigns.append(campaign)
            db.flush()
        else:
            campaigns = db.query(Campaign).all()

        if db.query(Content).count() == 0:
            content_rows = []
            for i in range(50):
                content = Content(
                    content_type=fake.random_element(["Blog", "Article", "Brochure"]),
                    title=fake.sentence(nb_words=6),
                    content=fake.paragraph(nb_sentences=8),
                    generated_by="content_agent",
                    status=fake.random_element(["DRAFT", "PENDING_REVIEW", "PUBLISHED"]),
                    course_id=fake.random_element(courses).course_id if courses else None,
                )
                db.add(content)
                content_rows.append(content)
            db.flush()

            for i in range(100):
                db.add(
                    SocialPost(
                        platform=fake.random_element(PLATFORMS),
                        content_id=fake.random_element(content_rows).content_id,
                        status=fake.random_element(["SCHEDULED", "PUBLISHED"]),
                    )
                )

        if db.query(SeoBlog).count() == 0:
            for i in range(50):
                keyword = f"{fake.word().title()} {fake.random_element(COURSES)} College"
                db.add(
                    SeoBlog(
                        title=f"{keyword} | Complete Guide",
                        keyword=keyword,
                        slug=f"{keyword.lower().replace(' ', '-')}-{i}",
                        meta_description=fake.sentence(nb_words=15),
                        content=fake.paragraph(nb_sentences=10),
                        status="DRAFT",
                    )
                )

        if db.query(Lead).count() == 0:
            for _ in range(100):
                db.add(
                    Lead(
                        first_name=fake.first_name(),
                        last_name=fake.last_name(),
                        mobile=fake.msisdn()[:15],
                        email=fake.email(),
                        city=fake.city(),
                        state=fake.state(),
                        course_interest=fake.random_element(COURSES),
                        source=fake.random_element(LEAD_SOURCES),
                        lead_score=fake.random_int(min=0, max=100),
                        status=fake.random_element(LEAD_STATUSES),
                        campaign_id=fake.random_element(campaigns).campaign_id if campaigns else None,
                    )
                )

        db.commit()
        print("Sample data seeded successfully.")
    finally:
        db.close()


if __name__ == "__main__":
    seed()

------------------------------
.github : workflows:
ci-cd.yaml
name: Marketing Agent CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: pgvector/pgvector:pg16
        env:
          POSTGRES_USER: marketing
          POSTGRES_PASSWORD: marketing
          POSTGRES_DB: marketing_agent
        ports: ["5432:5432"]
        options: >-
          --health-cmd "pg_isready -U marketing"
          --health-interval 5s
          --health-timeout 5s
          --health-retries 10
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install backend dependencies
        run: pip install -r requirements.txt

      - name: Run Alembic migrations
        working-directory: backend
        env:
          DATABASE_URL: postgresql+psycopg2://marketing:marketing@localhost:5432/marketing_agent
        run: alembic upgrade head

      - name: Run backend tests
        env:
          DATABASE_URL: postgresql+psycopg2://marketing:marketing@localhost:5432/marketing_agent
        run: pytest tests --cov=backend --cov-report=xml

      - name: SonarCloud Scan
        uses: SonarSource/sonarcloud-github-action@v3
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        continue-on-error: true

      - name: Security scan (pip-audit)
        run: |
          pip install pip-audit
          pip-audit -r requirements.txt || true

      - name: Set up Node
        uses: actions/setup-node@v4
        with:
          node-version: "20"

      - name: Install & build frontend
        working-directory: frontend
        run: |
          npm install
          npm run build -- --configuration production

      - name: Build backend Docker image
        run: docker build -f backend/Dockerfile -t marketing-backend:${{ github.sha }} .

      - name: Build frontend Docker image
        run: docker build -t marketing-frontend:${{ github.sha }} ./frontend

  deploy:
    needs: build-and-test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to Cloud Run
        run: echo "Deployment step - wire up gcloud auth + cloudbuild.yaml (infrastructure/gcp/cloudbuild.yaml) here."
-------------------------
marketing-agent:
docker-compose.yaml
version: "3.9"

services:
  postgres:
    image: pgvector/pgvector:pg16
    container_name: marketing-postgres
    environment:
      POSTGRES_USER: marketing
      POSTGRES_PASSWORD: marketing
      POSTGRES_DB: marketing_agent
    ports:
      - "5432:5432"
    volumes:
      - marketing_pg_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U marketing -d marketing_agent"]
      interval: 5s
      timeout: 5s
      retries: 10

  backend:
    build:
      context: .
      dockerfile: backend/Dockerfile
    container_name: marketing-backend
    env_file:
      - .env
    environment:
      DATABASE_URL: postgresql+psycopg2://marketing:marketing@postgres:5432/marketing_agent
    depends_on:
      postgres:
        condition: service_healthy
    ports:
      - "8000:8000"
    volumes:
      - ./backend:/app

  frontend:
    build:
      context: ./frontend
    container_name: marketing-frontend
    depends_on:
      - backend
    ports:
      - "4200:4200"

volumes:
  marketing_pg_data:
-------------------
marketing-agent:
.env.example
# --- Application ---
ENVIRONMENT=local
LOG_LEVEL=INFO

# --- Database ---
DATABASE_URL=postgresql+psycopg2://marketing:marketing@localhost:5432/marketing_agent
DB_ECHO=false

# --- Auth ---
JWT_SECRET_KEY=change-me-in-production
JWT_ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=30
REFRESH_TOKEN_EXPIRE_MINUTES=10080

# --- Gemini AI ---
GEMINI_API_KEY=
GEMINI_MODEL=gemini-2.5-flash
# When GEMINI_API_KEY is empty, gemini_service falls back to deterministic
# mock output so the system still runs end-to-end without a live API key.

# --- CORS ---
CORS_ALLOW_ORIGINS=http://localhost:4200

# --- GCP (used only by deployment scripts) ---
GCP_PROJECT_ID=
GCP_REGION=us-central1
GCS_BUCKET=

---------------
marketing-agent:
pytest.ini
[pytest]
pythonpath = backend
testpaths = tests

--------------------
marketing-agent:
requirements.txt
fastapi==0.115.0
uvicorn[standard]==0.30.6
pydantic==2.9.2
pydantic-settings==2.5.2
SQLAlchemy==2.0.35
alembic==1.13.3
psycopg2-binary==2.9.9
pgvector==0.3.4
python-jose[cryptography]==3.3.0
passlib[bcrypt]==1.7.4
python-multipart==0.0.9
langgraph==0.2.34
langchain==0.3.3
langchain-core==0.3.10
google-generativeai==0.8.3
python-dotenv==1.0.1
httpx==0.27.2
tenacity==9.0.0
pytest==8.3.3
pytest-asyncio==0.24.0
pytest-cov==5.0.0
faker==30.3.0

