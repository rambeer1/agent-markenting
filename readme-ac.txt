C:\ws\agent\admission\counselor-agent\backend\agents\assessment_agent\__init__.py

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\agents\assessment_agent\career_catalog.py
"""Static career criteria catalog used by the dimension scoring engine (Phase 8)
and the gap identification engine (Phase 10).

Market demand scores are placeholders until Phase 14 wires the real Job Market MCP.
"""

CAREER_DEFINITIONS = {
    "AIEngineer": {
        "required_skills": ["Python", "AI"],
        "target_skills": ["Python", "AI", "Data Analytics", "Cloud"],
        "interest_map": {"Interests": "D", "Strength Analysis": "A"},
        "academic_subjects": ["Math", "Computer Science"],
        "market_demand_score": 95.0,
    },
    "DataScientist": {
        "required_skills": ["SQL", "Python", "Data Analytics"],
        "target_skills": ["Python", "SQL", "Statistics", "Machine Learning", "Data Visualization", "Cloud Analytics"],
        "interest_map": {"Interests": "D", "Strength Analysis": "B"},
        "academic_subjects": ["Math", "Computer Science"],
        "market_demand_score": 90.0,
    },
    "CloudArchitect": {
        "required_skills": ["Cloud", "Problem Solving"],
        "target_skills": ["Cloud", "Problem Solving", "System Design"],
        "interest_map": {"Interests": "A", "Strength Analysis": "A"},
        "academic_subjects": ["Computer Science", "Physics"],
        "market_demand_score": 85.0,
    },
    "ProductManager": {
        "required_skills": ["Leadership", "Communication"],
        "target_skills": ["Leadership", "Communication", "Business Understanding"],
        "interest_map": {"Interests": "E", "Strength Analysis": "D"},
        "academic_subjects": ["English", "Business Studies"],
        "market_demand_score": 75.0,
    },
}

SOFT_SKILL_NAMES = {"Communication", "Leadership", "Presentation", "Problem Solving", "Collaboration", "Business Understanding"}


def infer_skill_type(skill_name: str) -> str:
    return "SOFT" if skill_name in SOFT_SKILL_NAMES else "TECHNICAL"



-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\agents\learning_agent\__init__.py

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\agents\mentor_agent\__init__.py

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\agents\orchestrator\__init__.py
C:\ws\agent\admission\counselor-agent\backend\agents\orchestrator\graph.py
from typing import TypedDict

from langgraph.checkpoint.memory import MemorySaver
from langgraph.graph import END, START, StateGraph

from database.session import SessionLocal
from repositories.academic_history_repository import list_academic_history
from repositories.assessment_repository import save_assessment_score
from repositories.learning_platform_repository import list_learning_records
from repositories.recommendation_repository import create_recommendation
from repositories.student_repository import get_student
from services.assessment_service import compute_scores
from services.gemini_service import GeminiServiceError, generate_text
from services.mentor_service import chat as mentor_chat
from services.progress_service import get_progress_for_student
from services.recommendation_service import generate_recommendations
from services.roadmap_service import generate_roadmap

MAX_RETRIES = 2


class OrchestratorState(TypedDict, total=False):
    input: str
    output: str
    error: str
    retry_count: int


def passthrough_node(state: OrchestratorState) -> OrchestratorState:
    """Single placeholder business node: round-trips `input` through Gemini."""
    try:
        response = generate_text(state["input"])
        return {"output": response, "error": None}
    except GeminiServiceError as exc:
        return {"retry_count": state.get("retry_count", 0) + 1, "error": str(exc)}


def _route_after_passthrough(state: OrchestratorState) -> str:
    if state.get("output"):
        return END
    if state.get("retry_count", 0) > MAX_RETRIES:
        return END
    return "passthrough"


def build_orchestrator_graph():
    graph = StateGraph(OrchestratorState)
    graph.add_node("passthrough", passthrough_node)
    graph.set_entry_point("passthrough")
    graph.add_conditional_edges("passthrough", _route_after_passthrough, {"passthrough": "passthrough", END: END})
    return graph.compile(checkpointer=MemorySaver())


orchestrator = build_orchestrator_graph()


def run(input_text: str, thread_id: str = "default") -> OrchestratorState:
    config = {"configurable": {"thread_id": thread_id}}
    return orchestrator.invoke({"input": input_text, "retry_count": 0}, config=config)


# --- Phase 15: full multi-agent counselor orchestration -----------------------------


class CounselorState(TypedDict, total=False):
    student_id: int
    career_names: list[str] | None
    mentor_message: str | None
    profile: dict | None
    profile_error: str | None
    learning_history: dict | None
    learning_error: str | None
    assessment_scores: list[dict] | None
    assessment_error: str | None
    recommendations: list[dict] | None
    top_recommendation_id: int | None
    recommendation_error: str | None
    roadmap: dict | None
    roadmap_error: str | None
    mentor_reply: dict | None
    mentor_error: str | None
    progress: list[dict] | None
    progress_error: str | None


def _with_retry(fn):
    """Runs fn() up to MAX_RETRIES+1 times, returning (result, error_message)."""
    last_error = None
    for _ in range(MAX_RETRIES + 1):
        try:
            return fn(), None
        except Exception as exc:  # noqa: BLE001 - a node must never crash the whole graph
            last_error = str(exc)
    return None, last_error


def profile_node(state: CounselorState) -> CounselorState:
    def _run():
        db = SessionLocal()
        try:
            student = get_student(db, state["student_id"])
            if not student:
                raise ValueError(f"Student {state['student_id']} not found")
            return {
                "student_id": student.student_id,
                "name": student.name,
                "age": student.age,
                "education": student.education,
                "subjects": student.subjects,
            }
        finally:
            db.close()

    result, error = _with_retry(_run)
    return {"profile": result, "profile_error": error}


def learning_node(state: CounselorState) -> CounselorState:
    def _run():
        db = SessionLocal()
        try:
            academic = list_academic_history(db, state["student_id"])
            learning = list_learning_records(db, state["student_id"])
            return {
                "academic_history": [{"subject": r.subject, "score": r.score} for r in academic],
                "learning_platform_data": [
                    {"platform": r.platform, "courses_completed": r.courses_completed} for r in learning
                ],
            }
        finally:
            db.close()

    result, error = _with_retry(_run)
    return {"learning_history": result, "learning_error": error}


def assessment_node(state: CounselorState) -> CounselorState:
    def _run():
        db = SessionLocal()
        try:
            results = compute_scores(db, state["student_id"], state.get("career_names"))
            for r in results:
                dimension_scores = {k: v for k, v in r.items() if k != "career_name"}
                save_assessment_score(db, state["student_id"], r["career_name"], dimension_scores)
            return results
        finally:
            db.close()

    result, error = _with_retry(_run)
    return {"assessment_scores": result, "assessment_error": error}


def recommendation_node(state: CounselorState) -> CounselorState:
    def _run():
        db = SessionLocal()
        try:
            results = generate_recommendations(db, state["student_id"], state.get("career_names"))
            records = [
                create_recommendation(db, state["student_id"], r["career_name"], r["match_score"], r["rationale"])
                for r in results
            ]
            records.sort(key=lambda r: r.match_score, reverse=True)
            summaries = [
                {
                    "recommendation_id": r.recommendation_id,
                    "career_name": r.career_name,
                    "match_score": r.match_score,
                }
                for r in records
            ]
            top_id = records[0].recommendation_id if records else None
            return summaries, top_id
        finally:
            db.close()

    result, error = _with_retry(_run)
    recommendations, top_id = result if result else (None, None)
    return {"recommendations": recommendations, "top_recommendation_id": top_id, "recommendation_error": error}


def roadmap_node(state: CounselorState) -> CounselorState:
    top_recommendation_id = state.get("top_recommendation_id")
    if not top_recommendation_id:
        return {"roadmap": None, "roadmap_error": "no recommendation available to build a roadmap from"}

    def _run():
        db = SessionLocal()
        try:
            built = generate_roadmap(db, top_recommendation_id)
            roadmap = built["roadmap"]
            return {
                "roadmap_id": roadmap.roadmap_id,
                "goal": roadmap.goal,
                "duration_months": roadmap.duration_months,
                "milestone_count": len(built["milestones"]),
            }
        finally:
            db.close()

    result, error = _with_retry(_run)
    return {"roadmap": result, "roadmap_error": error}


def mentor_node(state: CounselorState) -> CounselorState:
    message = state.get("mentor_message") or "What should I focus on next in my career journey?"

    def _run():
        db = SessionLocal()
        try:
            return mentor_chat(db, state["student_id"], message)
        finally:
            db.close()

    result, error = _with_retry(_run)
    return {"mentor_reply": result, "mentor_error": error}


def progress_node(state: CounselorState) -> CounselorState:
    def _run():
        db = SessionLocal()
        try:
            entries = get_progress_for_student(db, state["student_id"])
            return [
                {
                    "roadmap_id": e["progress"].roadmap_id,
                    "milestones_completed": e["progress"].milestones_completed,
                    "total_milestones": e["total_milestones"],
                    "completion_percentage": e["progress"].completion_percentage,
                }
                for e in entries
            ]
        finally:
            db.close()

    result, error = _with_retry(_run)
    return {"progress": result, "progress_error": error}


def build_counselor_graph():
    """Fan-out Profile/Learning/Assessment, fan-in to Recommendation, then linear Roadmap -> Mentor -> Progress."""
    graph = StateGraph(CounselorState)
    graph.add_node("profile", profile_node)
    graph.add_node("learning", learning_node)
    graph.add_node("assessment", assessment_node)
    graph.add_node("recommendation", recommendation_node)
    graph.add_node("roadmap", roadmap_node)
    graph.add_node("mentor", mentor_node)
    graph.add_node("progress", progress_node)

    graph.add_edge(START, "profile")
    graph.add_edge(START, "learning")
    graph.add_edge(START, "assessment")

    graph.add_edge("profile", "recommendation")
    graph.add_edge("learning", "recommendation")
    graph.add_edge("assessment", "recommendation")

    graph.add_edge("recommendation", "roadmap")
    graph.add_edge("roadmap", "mentor")
    graph.add_edge("mentor", "progress")
    graph.add_edge("progress", END)

    return graph.compile(checkpointer=MemorySaver())


counselor_graph = build_counselor_graph()


def run_counselor_flow(
    student_id: int,
    career_names: list[str] | None = None,
    mentor_message: str | None = None,
    thread_id: str = "default",
) -> CounselorState:
    config = {"configurable": {"thread_id": thread_id}}
    return counselor_graph.invoke(
        {"student_id": student_id, "career_names": career_names, "mentor_message": mentor_message},
        config=config,
    )

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\agents\profile_agent\__init__.py

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\agents\progress_tracking_agent\__init__.py

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\agents\recommendation_agent\__init__.py

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\agents\roadmap_agent\__init__.py

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\agents\__init__.py

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\api\dependencies\__init__.py
C:\ws\agent\admission\counselor-agent\backend\api\dependencies\auth.py
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPAuthorizationCredentials, HTTPBearer
from sqlalchemy.orm import Session

from database.session import get_db
from models.user import User
from repositories.user_repository import get_user_by_id
from security.jwt import ACCESS_TOKEN_TYPE, decode_token
from services.auth_service import is_token_blacklisted

bearer_scheme = HTTPBearer()


def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(bearer_scheme),
    db: Session = Depends(get_db),
) -> User:
    try:
        payload = decode_token(credentials.credentials)
    except ValueError as exc:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Invalid or expired token") from exc

    if payload.get("type") != ACCESS_TOKEN_TYPE:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Invalid token type")
    if is_token_blacklisted(db, payload["jti"]):
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "Token has been revoked")

    user = get_user_by_id(db, int(payload["sub"]))
    if not user or not user.is_active:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, "User not found or inactive")
    return user


def require_roles(*allowed_roles: str):
    def _check(current_user: User = Depends(get_current_user)) -> User:
        if current_user.role.role_name not in allowed_roles:
            raise HTTPException(status.HTTP_403_FORBIDDEN, "Insufficient permissions")
        return current_user

    return _check

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\api\routes\__init__.py
C:\ws\agent\admission\counselor-agent\backend\api\routes\academic_learning_history.py
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.orm import Session

from api.dependencies.auth import require_roles
from database.session import get_db
from models.user import User
from repositories.academic_history_repository import create_academic_history, list_academic_history
from repositories.learning_platform_repository import create_learning_record, list_learning_records
from repositories.student_repository import get_student
from schemas.academic_history import AcademicHistoryCreate, AcademicHistoryOut
from schemas.learning_platform_data import LearningPlatformDataCreate, LearningPlatformDataOut
from schemas.student_history import StudentHistoryOut
from security.roles import ADMIN, COUNSELOR, MENTOR, READ_ONLY, STUDENT

router = APIRouter(prefix="/api/v1/students", tags=["academic-learning-history"])

STAFF_ROLES = (ADMIN, COUNSELOR, MENTOR, READ_ONLY)
WRITE_ROLES = (ADMIN, COUNSELOR, MENTOR)


def _get_student_or_404(db: Session, student_id: int):
    student = get_student(db, student_id)
    if not student:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "Student not found")
    return student


@router.post(
    "/{student_id}/academic-history",
    response_model=AcademicHistoryOut,
    status_code=status.HTTP_201_CREATED,
)
def add_academic_history(
    student_id: int,
    payload: AcademicHistoryCreate,
    db: Session = Depends(get_db),
    _: User = Depends(require_roles(*WRITE_ROLES)),
):
    _get_student_or_404(db, student_id)
    return create_academic_history(db, student_id, payload.model_dump())


@router.post(
    "/{student_id}/learning-history",
    response_model=LearningPlatformDataOut,
    status_code=status.HTTP_201_CREATED,
)
def add_learning_history(
    student_id: int,
    payload: LearningPlatformDataCreate,
    db: Session = Depends(get_db),
    _: User = Depends(require_roles(*WRITE_ROLES)),
):
    _get_student_or_404(db, student_id)
    return create_learning_record(db, student_id, payload.model_dump())


@router.get("/{student_id}/history", response_model=StudentHistoryOut)
def get_student_history(
    student_id: int,
    db: Session = Depends(get_db),
    _: User = Depends(require_roles(*STAFF_ROLES, STUDENT)),
):
    _get_student_or_404(db, student_id)
    return StudentHistoryOut(
        academic_history=list_academic_history(db, student_id),
        learning_platform_data=list_learning_records(db, student_id),
    )

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\api\routes\ai_diagnostics.py
from fastapi import APIRouter, Depends
from pydantic import BaseModel

from agents.orchestrator.graph import run as run_orchestrator
from api.dependencies.auth import require_roles
from models.user import User
from security.roles import ADMIN

router = APIRouter(prefix="/api/v1/ai", tags=["ai-infra"])


class SamplePromptRequest(BaseModel):
    prompt: str


class SamplePromptResponse(BaseModel):
    output: str | None = None
    error: str | None = None


@router.post("/sample-prompt", response_model=SamplePromptResponse)
def sample_prompt(payload: SamplePromptRequest, _: User = Depends(require_roles(ADMIN))):
    """Diagnostic endpoint: round-trips a prompt through the orchestrator/Gemini (Phase 4 exit criteria)."""
    result = run_orchestrator(payload.prompt)
    return SamplePromptResponse(output=result.get("output"), error=result.get("error"))

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\api\routes\analytics.py
from fastapi import APIRouter, Depends
from sqlalchemy.orm import Session

from api.dependencies.auth import require_roles
from database.session import get_db
from models.user import User
from schemas.analytics import (
    CareerDemandTrendsOut,
    CertificationUptakeOut,
    DashboardSummaryOut,
    RoadmapCompletionRateOut,
    SkillGapHeatmapOut,
    TopRecommendedCareersOut,
)
from security.roles import ADMIN, COUNSELOR, MENTOR, READ_ONLY
from services.analytics_service import (
    career_demand_trends,
    certification_uptake,
    cohort_skill_gap_heatmap,
    dashboard_summary,
    roadmap_completion_rate,
    top_recommended_careers,
)

router = APIRouter(prefix="/api/v1/analytics", tags=["analytics"])

# Frontend grants all four counselor-portal roles (incl. READ_ONLY) access to /counselor/analytics.
READ_ROLES = (ADMIN, COUNSELOR, MENTOR, READ_ONLY)


@router.get("/career-demand-trends", response_model=CareerDemandTrendsOut)
def get_career_demand_trends(_: User = Depends(require_roles(*READ_ROLES))):
    return career_demand_trends()


@router.get("/skill-gap-heatmap", response_model=SkillGapHeatmapOut)
def get_skill_gap_heatmap(db: Session = Depends(get_db), _: User = Depends(require_roles(*READ_ROLES))):
    return cohort_skill_gap_heatmap(db)


@router.get("/roadmap-completion-rate", response_model=RoadmapCompletionRateOut)
def get_roadmap_completion_rate(db: Session = Depends(get_db), _: User = Depends(require_roles(*READ_ROLES))):
    return roadmap_completion_rate(db)


@router.get("/top-recommended-careers", response_model=TopRecommendedCareersOut)
def get_top_recommended_careers(db: Session = Depends(get_db), _: User = Depends(require_roles(*READ_ROLES))):
    return top_recommended_careers(db)


@router.get("/certification-uptake", response_model=CertificationUptakeOut)
def get_certification_uptake(db: Session = Depends(get_db), _: User = Depends(require_roles(*READ_ROLES))):
    return certification_uptake(db)


@router.get("/dashboard", response_model=DashboardSummaryOut)
def get_dashboard(db: Session = Depends(get_db), _: User = Depends(require_roles(*READ_ROLES))):
    return dashboard_summary(db)

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\api\routes\assessment.py
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.orm import Session

from api.dependencies.auth import require_roles
from database.session import get_db
from models.user import User
from repositories.assessment_repository import list_assessment_scores, save_assessment_score
from schemas.assessment import AssessmentRequest, AssessmentResponse, AssessmentScoreOut
from security.roles import ADMIN, COUNSELOR, MENTOR, READ_ONLY, STUDENT
from services.assessment_service import compute_scores

router = APIRouter(prefix="/api/v1/assessment", tags=["assessment"])

RUN_ROLES = (ADMIN, COUNSELOR, MENTOR)
READ_ROLES = (ADMIN, COUNSELOR, MENTOR, READ_ONLY, STUDENT)


@router.post("/score", response_model=AssessmentResponse, status_code=status.HTTP_201_CREATED)
def score(
    payload: AssessmentRequest,
    db: Session = Depends(get_db),
    _: User = Depends(require_roles(*RUN_ROLES)),
):
    try:
        results = compute_scores(db, payload.student_id, payload.career_names)
    except ValueError:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "Student not found")
    except KeyError as exc:
        raise HTTPException(status.HTTP_400_BAD_REQUEST, str(exc))

    details = []
    for r in results:
        career_name = r["career_name"]
        dimension_scores = {k: v for k, v in r.items() if k != "career_name"}
        record = save_assessment_score(db, payload.student_id, career_name, dimension_scores)
        details.append(AssessmentScoreOut.model_validate(record))
    scores = {r["career_name"]: r["final_score"] for r in results}
    return AssessmentResponse(scores=scores, details=details)


@router.get("/students/{student_id}/scores", response_model=list[AssessmentScoreOut])
def get_scores(
    student_id: int,
    db: Session = Depends(get_db),
    _: User = Depends(require_roles(*READ_ROLES)),
):
    return list_assessment_scores(db, student_id)

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\api\routes\auth.py
from fastapi import APIRouter, Depends, HTTPException, status
from fastapi.security import HTTPAuthorizationCredentials
from sqlalchemy.orm import Session

from api.dependencies.auth import bearer_scheme, get_current_user
from database.session import get_db
from models.user import User
from schemas.auth import (
    AccessTokenResponse,
    LoginRequest,
    PasswordResetConfirmRequest,
    PasswordResetRequest,
    PasswordResetRequestResponse,
    RefreshRequest,
    TokenResponse,
    UserOut,
)
from services.auth_service import (
    AuthError,
    authenticate,
    confirm_password_reset,
    issue_tokens,
    refresh_access_token,
    request_password_reset,
    revoke_token,
)

router = APIRouter(prefix="/api/v1/auth", tags=["auth"])


@router.post("/login", response_model=TokenResponse)
def login(payload: LoginRequest, db: Session = Depends(get_db)):
    try:
        user = authenticate(db, payload.email, payload.password)
    except AuthError as exc:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, str(exc)) from exc
    access_token, refresh_token = issue_tokens(user)
    return TokenResponse(access_token=access_token, refresh_token=refresh_token)


@router.post("/refresh", response_model=AccessTokenResponse)
def refresh(payload: RefreshRequest, db: Session = Depends(get_db)):
    try:
        access_token = refresh_access_token(db, payload.refresh_token)
    except (AuthError, ValueError) as exc:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, str(exc)) from exc
    return AccessTokenResponse(access_token=access_token)


@router.post("/logout", status_code=status.HTTP_204_NO_CONTENT)
def logout(
    credentials: HTTPAuthorizationCredentials = Depends(bearer_scheme),
    db: Session = Depends(get_db),
):
    try:
        revoke_token(db, credentials.credentials)
    except ValueError as exc:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, str(exc)) from exc


@router.post("/password-reset/request", response_model=PasswordResetRequestResponse)
def password_reset_request(payload: PasswordResetRequest, db: Session = Depends(get_db)):
    # Always responds 200 even for unknown emails, to avoid leaking registered addresses.
    reset_token = request_password_reset(db, payload.email)
    return PasswordResetRequestResponse(reset_token=reset_token or "")


@router.post("/password-reset/confirm", status_code=status.HTTP_204_NO_CONTENT)
def password_reset_confirm(payload: PasswordResetConfirmRequest, db: Session = Depends(get_db)):
    try:
        confirm_password_reset(db, payload.reset_token, payload.new_password)
    except (AuthError, ValueError) as exc:
        raise HTTPException(status.HTTP_401_UNAUTHORIZED, str(exc)) from exc


@router.get("/me", response_model=UserOut)
def me(current_user: User = Depends(get_current_user)):
    return UserOut(
        user_id=current_user.user_id,
        email=current_user.email,
        full_name=current_user.full_name,
        role=current_user.role.role_name,
        is_active=current_user.is_active,
        student_id=current_user.student_id,
    )

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\api\routes\catalog.py
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.orm import Session

from agents.assessment_agent.career_catalog import CAREER_DEFINITIONS
from api.dependencies.auth import require_roles
from database.session import get_db
from models.user import User
from repositories.catalog_repository import (
    create_certification,
    create_project,
    delete_certification,
    delete_project,
    get_certification,
    get_project,
    list_certifications,
    list_projects,
    update_certification,
    update_project,
)
from schemas.catalog import (
    CareerDefinitionOut,
    CertificationCreate,
    CertificationOut,
    CertificationUpdate,
    ProjectCreate,
    ProjectOut,
    ProjectUpdate,
)
from security.roles import ADMIN, COUNSELOR, MENTOR, READ_ONLY

router = APIRouter(prefix="/api/v1/catalog", tags=["catalog"])

READ_ROLES = (ADMIN, COUNSELOR, MENTOR, READ_ONLY)
WRITE_ROLES = (ADMIN, COUNSELOR)


@router.get("/careers", response_model=list[CareerDefinitionOut])
def list_careers(_: User = Depends(require_roles(*READ_ROLES))):
    # CAREER_DEFINITIONS is a static catalog consumed by the scoring/gap/roadmap
    # engines - exposed here read-only. Editing it would require migrating those
    # engines off the static dict, which is out of scope for this endpoint.
    return [
        CareerDefinitionOut(career_name=name, **definition) for name, definition in CAREER_DEFINITIONS.items()
    ]


@router.get("/certifications", response_model=list[CertificationOut])
def list_certifications_endpoint(
    db: Session = Depends(get_db),
    _: User = Depends(require_roles(*READ_ROLES)),
):
    return list_certifications(db)


@router.post("/certifications", response_model=CertificationOut, status_code=status.HTTP_201_CREATED)
def create_certification_endpoint(
    payload: CertificationCreate,
    db: Session = Depends(get_db),
    _: User = Depends(require_roles(*WRITE_ROLES)),
):
    return create_certification(db, payload.model_dump())


@router.put("/certifications/{certification_id}", response_model=CertificationOut)
def update_certification_endpoint(
    certification_id: int,
    payload: CertificationUpdate,
    db: Session = Depends(get_db),
    _: User = Depends(require_roles(*WRITE_ROLES)),
):
    certification = get_certification(db, certification_id)
    if not certification:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "Certification not found")
    return update_certification(db, certification, payload.model_dump(exclude_unset=True))


@router.delete("/certifications/{certification_id}", status_code=status.HTTP_204_NO_CONTENT)
def delete_certification_endpoint(
    certification_id: int,
    db: Session = Depends(get_db),
    _: User = Depends(require_roles(ADMIN)),
):
    certification = get_certification(db, certification_id)
    if not certification:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "Certification not found")
    delete_certification(db, certification)


@router.get("/projects", response_model=list[ProjectOut])
def list_projects_endpoint(
    db: Session = Depends(get_db),
    _: User = Depends(require_roles(*READ_ROLES)),
):
    return list_projects(db)


@router.post("/projects", response_model=ProjectOut, status_code=status.HTTP_201_CREATED)
def create_project_endpoint(
    payload: ProjectCreate,
    db: Session = Depends(get_db),
    _: User = Depends(require_roles(*WRITE_ROLES)),
):
    return create_project(db, payload.model_dump())


@router.put("/projects/{project_id}", response_model=ProjectOut)
def update_project_endpoint(
    project_id: int,
    payload: ProjectUpdate,
    db: Session = Depends(get_db),
    _: User = Depends(require_roles(*WRITE_ROLES)),
):
    project = get_project(db, project_id)
    if not project:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "Project not found")
    return update_project(db, project, payload.model_dump(exclude_unset=True))


@router.delete("/projects/{project_id}", status_code=status.HTTP_204_NO_CONTENT)
def delete_project_endpoint(
    project_id: int,
    db: Session = Depends(get_db),
    _: User = Depends(require_roles(ADMIN)),
):
    project = get_project(db, project_id)
    if not project:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "Project not found")
    delete_project(db, project)

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\api\routes\discovery.py
from fastapi import APIRouter, Depends, HTTPException, Query, Response, status
from sqlalchemy.orm import Session

from api.dependencies.auth import require_roles
from database.session import get_db
from models.user import User
from repositories.discovery_repository import (
    create_question,
    get_next_question,
    get_question,
    list_questions,
    record_answer,
    update_question,
)
from repositories.student_repository import get_student
from schemas.discovery import (
    DiscoveryAnswerRequest,
    DiscoveryQuestionCreate,
    DiscoveryQuestionOut,
    DiscoveryQuestionUpdate,
    DiscoveryResponseOut,
)
from security.roles import ADMIN, COUNSELOR, MENTOR, READ_ONLY, STUDENT

router = APIRouter(prefix="/api/v1/discovery", tags=["discovery"])

READ_ROLES = (ADMIN, COUNSELOR, MENTOR, READ_ONLY, STUDENT)
ANSWER_ROLES = (ADMIN, COUNSELOR, MENTOR, STUDENT)
BANK_READ_ROLES = (ADMIN, COUNSELOR, MENTOR, READ_ONLY)
BANK_WRITE_ROLES = (ADMIN, COUNSELOR)


@router.get("/next-question", response_model=DiscoveryQuestionOut | None)
def next_question(
    studentId: int = Query(...),
    db: Session = Depends(get_db),
    _: User = Depends(require_roles(*READ_ROLES)),
):
    if not get_student(db, studentId):
        raise HTTPException(status.HTTP_404_NOT_FOUND, "Student not found")

    question = get_next_question(db, studentId)
    if not question:
        return Response(status_code=status.HTTP_204_NO_CONTENT)
    return question


@router.post("/answer", response_model=DiscoveryResponseOut, status_code=status.HTTP_201_CREATED)
def answer(
    payload: DiscoveryAnswerRequest,
    db: Session = Depends(get_db),
    _: User = Depends(require_roles(*ANSWER_ROLES)),
):
    if not get_student(db, payload.student_id):
        raise HTTPException(status.HTTP_404_NOT_FOUND, "Student not found")

    question = get_question(db, payload.question_id)
    if not question:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "Question not found")
    if payload.selected_option not in question.options:
        raise HTTPException(status.HTTP_400_BAD_REQUEST, "Invalid option for this question")

    return record_answer(db, payload.student_id, payload.question_id, payload.selected_option)


@router.get("/questions", response_model=list[DiscoveryQuestionOut])
def list_question_bank(
    db: Session = Depends(get_db),
    _: User = Depends(require_roles(*BANK_READ_ROLES)),
):
    return list_questions(db)


@router.post("/questions", response_model=DiscoveryQuestionOut, status_code=status.HTTP_201_CREATED)
def create_question_bank_entry(
    payload: DiscoveryQuestionCreate,
    db: Session = Depends(get_db),
    _: User = Depends(require_roles(*BANK_WRITE_ROLES)),
):
    return create_question(db, payload.category, payload.question_text, payload.options)


@router.put("/questions/{question_id}", response_model=DiscoveryQuestionOut)
def update_question_bank_entry(
    question_id: int,
    payload: DiscoveryQuestionUpdate,
    db: Session = Depends(get_db),
    _: User = Depends(require_roles(*BANK_WRITE_ROLES)),
):
    question = get_question(db, question_id)
    if not question:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "Question not found")
    return update_question(db, question, payload.model_dump(exclude_unset=True))

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\api\routes\gap_analysis.py
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.orm import Session

from api.dependencies.auth import require_roles
from database.session import get_db
from models.user import User
from schemas.gap_analysis import GapAnalysisRequest, GapAnalysisResponse, SkillGapOut
from security.roles import ADMIN, COUNSELOR, MENTOR, READ_ONLY, STUDENT
from services.gap_analysis_service import MATCHED, analyze_gap, get_gap_analysis

router = APIRouter(prefix="/api/v1/gap-analysis", tags=["gap-analysis"])

RUN_ROLES = (ADMIN, COUNSELOR, MENTOR)
READ_ROLES = (ADMIN, COUNSELOR, MENTOR, READ_ONLY, STUDENT)


def _to_response(recommendation_id: int, results: list[dict]) -> GapAnalysisResponse:
    matched = [r["skill_name"] for r in results if r["status"] == MATCHED]
    missing = [r["skill_name"] for r in results if r["status"] != MATCHED]
    details = [SkillGapOut(**r) for r in results]
    return GapAnalysisResponse(recommendation_id=recommendation_id, matched=matched, missing=missing, details=details)


@router.post("", response_model=GapAnalysisResponse, status_code=status.HTTP_201_CREATED)
def create_gap_analysis(
    payload: GapAnalysisRequest,
    db: Session = Depends(get_db),
    _: User = Depends(require_roles(*RUN_ROLES)),
):
    try:
        results = analyze_gap(db, payload.recommendation_id)
    except ValueError:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "Recommendation not found")
    except KeyError as exc:
        raise HTTPException(status.HTTP_400_BAD_REQUEST, str(exc))

    return _to_response(payload.recommendation_id, results)


@router.get("/{recommendation_id}", response_model=GapAnalysisResponse)
def read_gap_analysis(
    recommendation_id: int,
    db: Session = Depends(get_db),
    _: User = Depends(require_roles(*READ_ROLES)),
):
    results = get_gap_analysis(db, recommendation_id)
    return _to_response(recommendation_id, results)

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\api\routes\health.py
from fastapi import APIRouter

router = APIRouter(tags=["health"])


@router.get("/health")
def health_check():
    return {"status": "ok"}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\api\routes\mentor.py
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.orm import Session

from api.dependencies.auth import require_roles
from database.session import get_db
from models.user import User
from repositories.mentor_repository import list_conversations
from schemas.mentor import MentorChatRequest, MentorChatResponse, MentorConversationOut
from security.roles import ADMIN, COUNSELOR, MENTOR, READ_ONLY, STUDENT
from services.mentor_service import chat as run_chat

router = APIRouter(prefix="/api/v1/mentor", tags=["mentor"])

CHAT_ROLES = (ADMIN, COUNSELOR, MENTOR, STUDENT)
READ_ROLES = (ADMIN, COUNSELOR, MENTOR, READ_ONLY, STUDENT)


@router.post("/chat", response_model=MentorChatResponse, status_code=status.HTTP_201_CREATED)
def chat(
    payload: MentorChatRequest,
    db: Session = Depends(get_db),
    _: User = Depends(require_roles(*CHAT_ROLES)),
):
    try:
        result = run_chat(db, payload.student_id, payload.message)
    except ValueError:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "Student not found")

    return MentorChatResponse(reply=result["reply"], related_career=result["related_career"])


@router.get("/students/{student_id}", response_model=list[MentorConversationOut])
def get_history(
    student_id: int,
    db: Session = Depends(get_db),
    _: User = Depends(require_roles(*READ_ROLES)),
):
    return list_conversations(db, student_id)

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\api\routes\orchestrator.py
import uuid

from fastapi import APIRouter, Depends

from agents.orchestrator.graph import run_counselor_flow
from api.dependencies.auth import require_roles
from models.user import User
from schemas.orchestrator import CounselorFlowRequest, CounselorFlowResponse
from security.roles import ADMIN, COUNSELOR, MENTOR

router = APIRouter(prefix="/api/v1/orchestrator", tags=["orchestrator"])

RUN_ROLES = (ADMIN, COUNSELOR, MENTOR)


@router.post("/run", response_model=CounselorFlowResponse)
def run_flow(
    payload: CounselorFlowRequest,
    _: User = Depends(require_roles(*RUN_ROLES)),
):
    """Runs the full Profile/Learning/Assessment -> Recommendation -> Roadmap -> Mentor -> Progress flow."""
    result = run_counselor_flow(
        payload.student_id,
        payload.career_names,
        payload.mentor_message,
        thread_id=str(uuid.uuid4()),
    )
    return CounselorFlowResponse(**result)


-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\api\routes\progress.py
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.orm import Session

from api.dependencies.auth import require_roles
from database.session import get_db
from models.user import User
from schemas.progress import (
    ProgressCompleteResponse,
    ProgressMilestoneCompleteRequest,
    ProgressOut,
    UpdatedRecommendationOut,
)
from security.roles import ADMIN, COUNSELOR, MENTOR, READ_ONLY, STUDENT
from services.progress_service import complete_milestone, get_progress_for_student

router = APIRouter(prefix="/api/v1/progress", tags=["progress"])

RUN_ROLES = (ADMIN, COUNSELOR, MENTOR)
READ_ROLES = (ADMIN, COUNSELOR, MENTOR, READ_ONLY, STUDENT)


@router.post("/milestone/complete", response_model=ProgressCompleteResponse)
def milestone_complete(
    payload: ProgressMilestoneCompleteRequest,
    db: Session = Depends(get_db),
    _: User = Depends(require_roles(*RUN_ROLES)),
):
    try:
        result = complete_milestone(db, payload.milestone_id)
    except ValueError:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "Milestone not found")

    progress = result["progress"]
    updated_recommendation = (
        UpdatedRecommendationOut.model_validate(result["updated_recommendation"])
        if result["updated_recommendation"]
        else None
    )
    return ProgressCompleteResponse(
        tracking_id=progress.tracking_id,
        student_id=progress.student_id,
        roadmap_id=progress.roadmap_id,
        milestones_completed=progress.milestones_completed,
        total_milestones=result["total_milestones"],
        completion_percentage=progress.completion_percentage,
        last_reassessed_date=progress.last_reassessed_date,
        reassessment_triggered=result["reassessment_triggered"],
        updated_recommendation=updated_recommendation,
    )


@router.get("/{student_id}", response_model=list[ProgressOut])
def get_progress(
    student_id: int,
    db: Session = Depends(get_db),
    _: User = Depends(require_roles(*READ_ROLES)),
):
    entries = get_progress_for_student(db, student_id)
    return [
        ProgressOut(
            tracking_id=e["progress"].tracking_id,
            student_id=e["progress"].student_id,
            roadmap_id=e["progress"].roadmap_id,
            milestones_completed=e["progress"].milestones_completed,
            total_milestones=e["total_milestones"],
            completion_percentage=e["progress"].completion_percentage,
            last_reassessed_date=e["progress"].last_reassessed_date,
        )
        for e in entries
    ]

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\api\routes\prompts.py
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.orm import Session

from api.dependencies.auth import require_roles
from database.session import get_db
from models.user import User
from repositories.prompt_repository import create_prompt, delete_prompt, get_prompt, list_prompts
from schemas.prompt import PromptCreate, PromptOut
from security.roles import ADMIN, COUNSELOR

router = APIRouter(prefix="/api/v1/prompts", tags=["prompts"])

READ_ROLES = (ADMIN, COUNSELOR)
WRITE_ROLES = (ADMIN,)


@router.post("", response_model=PromptOut, status_code=status.HTTP_201_CREATED)
def create(
    payload: PromptCreate,
    db: Session = Depends(get_db),
    _: User = Depends(require_roles(*WRITE_ROLES)),
):
    return create_prompt(db, payload.agent_name, payload.prompt_type, payload.prompt_text)


@router.get("", response_model=list[PromptOut])
def list_all(
    agent_name: str | None = None,
    db: Session = Depends(get_db),
    _: User = Depends(require_roles(*READ_ROLES)),
):
    return list_prompts(db, agent_name)


@router.get("/{prompt_id}", response_model=PromptOut)
def get_one(
    prompt_id: int,
    db: Session = Depends(get_db),
    _: User = Depends(require_roles(*READ_ROLES)),
):
    prompt = get_prompt(db, prompt_id)
    if not prompt:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "Prompt not found")
    return prompt


@router.delete("/{prompt_id}", status_code=status.HTTP_204_NO_CONTENT)
def delete(
    prompt_id: int,
    db: Session = Depends(get_db),
    _: User = Depends(require_roles(*WRITE_ROLES)),
):
    prompt = get_prompt(db, prompt_id)
    if not prompt:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "Prompt not found")
    delete_prompt(db, prompt)

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\api\routes\recommendations.py
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.orm import Session

from api.dependencies.auth import require_roles
from database.session import get_db
from models.user import User
from repositories.recommendation_repository import create_recommendation, list_recommendations
from schemas.recommendation import RecommendationOut, RecommendationRequest, RecommendationResponse
from security.roles import ADMIN, COUNSELOR, MENTOR, READ_ONLY, STUDENT
from services.recommendation_service import generate_recommendations

router = APIRouter(prefix="/api/v1/recommendations", tags=["recommendations"])

RUN_ROLES = (ADMIN, COUNSELOR, MENTOR)
READ_ROLES = (ADMIN, COUNSELOR, MENTOR, READ_ONLY, STUDENT)


@router.post("/generate", response_model=RecommendationResponse, status_code=status.HTTP_201_CREATED)
def generate(
    payload: RecommendationRequest,
    db: Session = Depends(get_db),
    _: User = Depends(require_roles(*RUN_ROLES)),
):
    try:
        results = generate_recommendations(db, payload.student_id, payload.career_names, payload.top_n)
    except ValueError:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "Student not found")
    except KeyError as exc:
        raise HTTPException(status.HTTP_400_BAD_REQUEST, str(exc))

    recommendations = [
        RecommendationOut.model_validate(
            create_recommendation(db, payload.student_id, r["career_name"], r["match_score"], r["rationale"])
        )
        for r in results
    ]
    return RecommendationResponse(recommendations=recommendations)


@router.get("/students/{student_id}", response_model=list[RecommendationOut])
def get_recommendations(
    student_id: int,
    db: Session = Depends(get_db),
    _: User = Depends(require_roles(*READ_ROLES)),
):
    return list_recommendations(db, student_id)

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\api\routes\roadmaps.py
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.orm import Session

from api.dependencies.auth import require_roles
from database.session import get_db
from models.user import User
from repositories.roadmap_repository import list_roadmaps
from schemas.roadmap import RoadmapGenerateRequest, RoadmapOut
from security.roles import ADMIN, COUNSELOR, MENTOR, READ_ONLY, STUDENT
from services.roadmap_service import generate_roadmap, get_roadmap_with_milestones

router = APIRouter(prefix="/api/v1/roadmaps", tags=["roadmaps"])

RUN_ROLES = (ADMIN, COUNSELOR, MENTOR)
READ_ROLES = (ADMIN, COUNSELOR, MENTOR, READ_ONLY, STUDENT)


def _to_out(roadmap, milestones) -> RoadmapOut:
    return RoadmapOut(
        roadmap_id=roadmap.roadmap_id,
        student_id=roadmap.student_id,
        recommendation_id=roadmap.recommendation_id,
        goal=roadmap.goal,
        duration_months=roadmap.duration_months,
        status=roadmap.status,
        created_date=roadmap.created_date,
        milestones=milestones,
    )


@router.post("/generate", response_model=RoadmapOut, status_code=status.HTTP_201_CREATED)
def generate(
    payload: RoadmapGenerateRequest,
    db: Session = Depends(get_db),
    _: User = Depends(require_roles(*RUN_ROLES)),
):
    try:
        result = generate_roadmap(db, payload.recommendation_id, payload.duration_months)
    except ValueError:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "Recommendation not found")
    except KeyError as exc:
        raise HTTPException(status.HTTP_400_BAD_REQUEST, str(exc))

    return _to_out(result["roadmap"], result["milestones"])


@router.get("/{roadmap_id}", response_model=RoadmapOut)
def get_one(
    roadmap_id: int,
    db: Session = Depends(get_db),
    _: User = Depends(require_roles(*READ_ROLES)),
):
    result = get_roadmap_with_milestones(db, roadmap_id)
    if not result:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "Roadmap not found")
    return _to_out(result["roadmap"], result["milestones"])


@router.get("/students/{student_id}", response_model=list[RoadmapOut])
def list_for_student(
    student_id: int,
    db: Session = Depends(get_db),
    _: User = Depends(require_roles(*READ_ROLES)),
):
    roadmaps = list_roadmaps(db, student_id)
    return [_to_out(r, get_roadmap_with_milestones(db, r.roadmap_id)["milestones"]) for r in roadmaps]

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\api\routes\skills.py
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.orm import Session

from api.dependencies.auth import require_roles
from database.session import get_db
from models.user import User
from repositories.skill_repository import (
    get_or_create_skill,
    list_behavioral_metrics,
    list_student_skills,
    record_behavioral_metric,
    record_student_skill,
)
from repositories.student_repository import get_student
from schemas.skill_assessment import (
    BehavioralMetricOut,
    SkillAssessmentOut,
    SkillAssessmentRequest,
    StudentSkillOut,
)
from security.roles import ADMIN, COUNSELOR, MENTOR, READ_ONLY, STUDENT

router = APIRouter(prefix="/api/v1/students", tags=["skill-assessment"])

STAFF_ROLES = (ADMIN, COUNSELOR, MENTOR, READ_ONLY)
WRITE_ROLES = (ADMIN, COUNSELOR, MENTOR)


def _get_student_or_404(db: Session, student_id: int):
    student = get_student(db, student_id)
    if not student:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "Student not found")
    return student


def _to_student_skill_out(pair) -> StudentSkillOut:
    student_skill, skill = pair
    return StudentSkillOut(
        id=student_skill.id,
        student_id=student_skill.student_id,
        skill_id=skill.skill_id,
        skill_name=skill.skill_name,
        skill_type=skill.skill_type,
        proficiency_score=student_skill.proficiency_score,
        assessed_date=student_skill.assessed_date,
    )


@router.post("/{student_id}/skills/assess", response_model=SkillAssessmentOut, status_code=status.HTTP_201_CREATED)
def assess_skills(
    student_id: int,
    payload: SkillAssessmentRequest,
    db: Session = Depends(get_db),
    _: User = Depends(require_roles(*WRITE_ROLES)),
):
    _get_student_or_404(db, student_id)

    skill_results = []
    for item in payload.skills:
        skill = get_or_create_skill(db, item.skill_name, item.skill_type)
        record = record_student_skill(db, student_id, skill.skill_id, item.proficiency_score)
        skill_results.append(_to_student_skill_out((record, skill)))

    behavioral_out = None
    if payload.behavioral_metrics is not None:
        metric = record_behavioral_metric(
            db, student_id, payload.behavioral_metrics.model_dump(exclude_unset=True)
        )
        behavioral_out = BehavioralMetricOut.model_validate(metric)

    return SkillAssessmentOut(skills=skill_results, behavioral_metric=behavioral_out)


@router.get("/{student_id}/skills", response_model=list[StudentSkillOut])
def get_skills(
    student_id: int,
    db: Session = Depends(get_db),
    _: User = Depends(require_roles(*STAFF_ROLES, STUDENT)),
):
    _get_student_or_404(db, student_id)
    return [_to_student_skill_out(pair) for pair in list_student_skills(db, student_id)]


@router.get("/{student_id}/behavioral-metrics", response_model=list[BehavioralMetricOut])
def get_behavioral_metrics(
    student_id: int,
    db: Session = Depends(get_db),
    _: User = Depends(require_roles(*STAFF_ROLES, STUDENT)),
):
    _get_student_or_404(db, student_id)
    return list_behavioral_metrics(db, student_id)

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\api\routes\students.py
from fastapi import APIRouter, Depends, HTTPException, Query, status
from sqlalchemy.orm import Session

from api.dependencies.auth import require_roles
from database.session import get_db
from models.user import User
from repositories.student_repository import (
    create_student,
    delete_student,
    get_student,
    list_students,
    update_student,
)
from schemas.student import StudentCreate, StudentOut, StudentUpdate
from security.roles import ADMIN, COUNSELOR, MENTOR, READ_ONLY, STUDENT

router = APIRouter(prefix="/api/v1/students", tags=["students"])

STAFF_ROLES = (ADMIN, COUNSELOR, MENTOR, READ_ONLY)
WRITE_ROLES = (ADMIN, COUNSELOR)


@router.post("", response_model=StudentOut, status_code=status.HTTP_201_CREATED)
def create(
    payload: StudentCreate,
    db: Session = Depends(get_db),
    _: User = Depends(require_roles(*WRITE_ROLES)),
):
    return create_student(db, payload.model_dump())


@router.get("", response_model=list[StudentOut])
def list_all(
    skip: int = Query(default=0, ge=0),
    limit: int = Query(default=50, ge=1, le=200),
    db: Session = Depends(get_db),
    _: User = Depends(require_roles(*STAFF_ROLES)),
):
    return list_students(db, skip, limit)


@router.get("/{student_id}", response_model=StudentOut)
def get_one(
    student_id: int,
    db: Session = Depends(get_db),
    _: User = Depends(require_roles(*STAFF_ROLES, STUDENT)),
):
    student = get_student(db, student_id)
    if not student:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "Student not found")
    return student


@router.put("/{student_id}", response_model=StudentOut)
def update(
    student_id: int,
    payload: StudentUpdate,
    db: Session = Depends(get_db),
    _: User = Depends(require_roles(*WRITE_ROLES)),
):
    student = get_student(db, student_id)
    if not student:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "Student not found")
    return update_student(db, student, payload.model_dump(exclude_unset=True))


@router.delete("/{student_id}", status_code=status.HTTP_204_NO_CONTENT)
def delete(
    student_id: int,
    db: Session = Depends(get_db),
    _: User = Depends(require_roles(ADMIN)),
):
    student = get_student(db, student_id)
    if not student:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "Student not found")
    delete_student(db, student)

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\api\routes\users.py
from fastapi import APIRouter, Depends
from sqlalchemy.orm import Session

from api.dependencies.auth import require_roles
from database.session import get_db
from models.user import User
from schemas.auth import UserOut
from security.roles import ADMIN

router = APIRouter(prefix="/api/v1/users", tags=["users"])


@router.get("", response_model=list[UserOut])
def list_users(db: Session = Depends(get_db), _: User = Depends(require_roles(ADMIN))):
    users = db.query(User).all()
    return [
        UserOut(
            user_id=u.user_id,
            email=u.email,
            full_name=u.full_name,
            role=u.role.role_name,
            is_active=u.is_active,
        )
        for u in users
    ]

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\api\__init__.py

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\config\__init__.py
C:\ws\agent\admission\counselor-agent\backend\config\settings.py
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

    environment: str = "development"
    database_url: str = "postgresql://counselor:changeme@localhost:5432/counselor_agent"

    jwt_secret: str = "change-me-in-production"
    jwt_algorithm: str = "HS256"
    access_token_expire_minutes: int = 30
    refresh_token_expire_days: int = 7

    gemini_api_key: str = ""
    # Active model - switch by changing GEMINI_MODEL (must be one of gemini_available_models).
    gemini_model: str = "gemini-2.5-flash"
    # Comma-separated list of models selectable via gemini_model, for validation/reference.
    gemini_available_models: str = "gemini-2.5-flash,gemini-2.5-pro,gemini-2.0-flash,gemini-1.5-pro"
    gemini_temperature: float = 0.7
    # Approximate word-based cap on prompt size sent to the LLM (truncated if exceeded).
    gemini_max_input_words: int = 3000
    # Hard cap on generated response length, enforced via the SDK's generation_config.
    gemini_max_output_tokens: int = 1024

    @property
    def gemini_available_models_list(self) -> list[str]:
        return [model.strip() for model in self.gemini_available_models.split(",") if model.strip()]


settings = Settings()

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\database\migrations\versions\.gitkeep
C:\ws\agent\admission\counselor-agent\backend\database\migrations\versions\0001_initial_schema.py
"""initial schema

Revision ID: 0001_initial_schema
Revises:
Create Date: 2026-08-01

"""
from typing import Sequence, Union

from alembic import op
import sqlalchemy as sa

revision: str = "0001_initial_schema"
down_revision: Union[str, None] = None
branch_labels: Union[str, Sequence[str], None] = None
depends_on: Union[str, Sequence[str], None] = None


def upgrade() -> None:
    op.create_table(
        "students",
        sa.Column("student_id", sa.Integer(), primary_key=True, autoincrement=True),
        sa.Column("name", sa.String(length=150), nullable=False),
        sa.Column("age", sa.Integer(), nullable=False),
        sa.Column("education", sa.String(length=150), nullable=False),
        sa.Column("location", sa.String(length=150), nullable=True),
        sa.Column("board", sa.String(length=50), nullable=True),
        sa.Column("subjects", sa.JSON(), nullable=True),
        sa.Column("created_date", sa.DateTime(), nullable=False),
    )

    op.create_table(
        "skills",
        sa.Column("skill_id", sa.Integer(), primary_key=True, autoincrement=True),
        sa.Column("skill_name", sa.String(length=100), nullable=False, unique=True),
        sa.Column("skill_type", sa.String(length=20), nullable=False),
    )

    op.create_table(
        "discovery_questions",
        sa.Column("question_id", sa.Integer(), primary_key=True, autoincrement=True),
        sa.Column("category", sa.String(length=50), nullable=False),
        sa.Column("question_text", sa.String(length=500), nullable=False),
        sa.Column("options", sa.JSON(), nullable=False),
        sa.Column("version", sa.Integer(), nullable=False, server_default="1"),
    )

    op.create_table(
        "prompts",
        sa.Column("prompt_id", sa.Integer(), primary_key=True, autoincrement=True),
        sa.Column("agent_name", sa.String(length=100), nullable=False),
        sa.Column("prompt_type", sa.String(length=100), nullable=False),
        sa.Column("prompt_text", sa.Text(), nullable=False),
        sa.Column("version", sa.Integer(), nullable=False, server_default="1"),
    )

    op.create_table(
        "certifications",
        sa.Column("certification_id", sa.Integer(), primary_key=True, autoincrement=True),
        sa.Column("career_name", sa.String(length=100), nullable=False),
        sa.Column("certification_name", sa.String(length=255), nullable=False),
        sa.Column("provider", sa.String(length=100), nullable=True),
    )

    op.create_table(
        "projects",
        sa.Column("project_id", sa.Integer(), primary_key=True, autoincrement=True),
        sa.Column("career_name", sa.String(length=100), nullable=False),
        sa.Column("project_name", sa.String(length=255), nullable=False),
        sa.Column("description", sa.String(length=1000), nullable=True),
        sa.Column("difficulty_level", sa.String(length=20), nullable=True),
    )

    op.create_table(
        "academic_history",
        sa.Column("history_id", sa.Integer(), primary_key=True, autoincrement=True),
        sa.Column("student_id", sa.Integer(), sa.ForeignKey("students.student_id"), nullable=False),
        sa.Column("subject", sa.String(length=100), nullable=False),
        sa.Column("score", sa.Float(), nullable=False),
        sa.Column("attendance", sa.Float(), nullable=True),
        sa.Column("learning_behavior", sa.String(length=255), nullable=True),
        sa.Column("recorded_date", sa.DateTime(), nullable=False),
    )

    op.create_table(
        "learning_platform_data",
        sa.Column("record_id", sa.Integer(), primary_key=True, autoincrement=True),
        sa.Column("student_id", sa.Integer(), sa.ForeignKey("students.student_id"), nullable=False),
        sa.Column("platform", sa.String(length=100), nullable=False),
        sa.Column("courses_completed", sa.Integer(), nullable=False, server_default="0"),
        sa.Column("hours_spent", sa.Float(), nullable=False, server_default="0"),
        sa.Column("skill_area", sa.String(length=100), nullable=True),
        sa.Column("recorded_date", sa.DateTime(), nullable=False),
    )

    op.create_table(
        "student_skills",
        sa.Column("id", sa.Integer(), primary_key=True, autoincrement=True),
        sa.Column("student_id", sa.Integer(), sa.ForeignKey("students.student_id"), nullable=False),
        sa.Column("skill_id", sa.Integer(), sa.ForeignKey("skills.skill_id"), nullable=False),
        sa.Column("proficiency_score", sa.Float(), nullable=False),
        sa.Column("assessed_date", sa.DateTime(), nullable=False),
    )

    op.create_table(
        "behavioral_metrics",
        sa.Column("metric_id", sa.Integer(), primary_key=True, autoincrement=True),
        sa.Column("student_id", sa.Integer(), sa.ForeignKey("students.student_id"), nullable=False),
        sa.Column("learning_speed", sa.Float(), nullable=True),
        sa.Column("consistency", sa.Float(), nullable=True),
        sa.Column("curiosity_index", sa.Float(), nullable=True),
        sa.Column("persistence", sa.Float(), nullable=True),
        sa.Column("goal_completion_rate", sa.Float(), nullable=True),
        sa.Column("recorded_date", sa.DateTime(), nullable=False),
    )

    op.create_table(
        "discovery_responses",
        sa.Column("response_id", sa.Integer(), primary_key=True, autoincrement=True),
        sa.Column("student_id", sa.Integer(), sa.ForeignKey("students.student_id"), nullable=False),
        sa.Column(
            "question_id", sa.Integer(), sa.ForeignKey("discovery_questions.question_id"), nullable=False
        ),
        sa.Column("selected_option", sa.String(length=10), nullable=False),
        sa.Column("answered_date", sa.DateTime(), nullable=False),
    )

    op.create_table(
        "assessment_scores",
        sa.Column("score_id", sa.Integer(), primary_key=True, autoincrement=True),
        sa.Column("student_id", sa.Integer(), sa.ForeignKey("students.student_id"), nullable=False),
        sa.Column("career_name", sa.String(length=100), nullable=False),
        sa.Column("interest_score", sa.Float(), nullable=False, server_default="0"),
        sa.Column("skill_score", sa.Float(), nullable=False, server_default="0"),
        sa.Column("academic_score", sa.Float(), nullable=False, server_default="0"),
        sa.Column("aptitude_score", sa.Float(), nullable=False, server_default="0"),
        sa.Column("market_demand_score", sa.Float(), nullable=False, server_default="0"),
        sa.Column("final_score", sa.Float(), nullable=False, server_default="0"),
        sa.Column("computed_date", sa.DateTime(), nullable=False),
    )

    op.create_table(
        "career_recommendations",
        sa.Column("recommendation_id", sa.Integer(), primary_key=True, autoincrement=True),
        sa.Column("student_id", sa.Integer(), sa.ForeignKey("students.student_id"), nullable=False),
        sa.Column("career_name", sa.String(length=100), nullable=False),
        sa.Column("match_score", sa.Float(), nullable=False),
        sa.Column("rationale", sa.String(length=1000), nullable=True),
        sa.Column("created_date", sa.DateTime(), nullable=False),
    )

    op.create_table(
        "skill_gaps",
        sa.Column("gap_id", sa.Integer(), primary_key=True, autoincrement=True),
        sa.Column(
            "recommendation_id",
            sa.Integer(),
            sa.ForeignKey("career_recommendations.recommendation_id"),
            nullable=False,
        ),
        sa.Column("skill_id", sa.Integer(), sa.ForeignKey("skills.skill_id"), nullable=False),
        sa.Column("status", sa.String(length=10), nullable=False),
    )

    op.create_table(
        "roadmaps",
        sa.Column("roadmap_id", sa.Integer(), primary_key=True, autoincrement=True),
        sa.Column("student_id", sa.Integer(), sa.ForeignKey("students.student_id"), nullable=False),
        sa.Column(
            "recommendation_id",
            sa.Integer(),
            sa.ForeignKey("career_recommendations.recommendation_id"),
            nullable=False,
        ),
        sa.Column("goal", sa.String(length=150), nullable=False),
        sa.Column("duration_months", sa.Integer(), nullable=False),
        sa.Column("status", sa.String(length=20), nullable=False, server_default="ACTIVE"),
        sa.Column("created_date", sa.DateTime(), nullable=False),
    )

    op.create_table(
        "roadmap_milestones",
        sa.Column("milestone_id", sa.Integer(), primary_key=True, autoincrement=True),
        sa.Column("roadmap_id", sa.Integer(), sa.ForeignKey("roadmaps.roadmap_id"), nullable=False),
        sa.Column("month_range", sa.String(length=20), nullable=False),
        sa.Column("topics", sa.JSON(), nullable=False),
        sa.Column("resources", sa.JSON(), nullable=True),
        sa.Column("project", sa.String(length=255), nullable=True),
        sa.Column("status", sa.String(length=20), nullable=False, server_default="PENDING"),
        sa.Column("completed_date", sa.DateTime(), nullable=True),
    )

    op.create_table(
        "mentor_conversations",
        sa.Column("conversation_id", sa.Integer(), primary_key=True, autoincrement=True),
        sa.Column("student_id", sa.Integer(), sa.ForeignKey("students.student_id"), nullable=False),
        sa.Column("message", sa.String(length=2000), nullable=False),
        sa.Column("reply", sa.String(length=2000), nullable=True),
        sa.Column("related_career", sa.String(length=100), nullable=True),
        sa.Column("created_date", sa.DateTime(), nullable=False),
    )

    op.create_table(
        "progress_tracking",
        sa.Column("tracking_id", sa.Integer(), primary_key=True, autoincrement=True),
        sa.Column("student_id", sa.Integer(), sa.ForeignKey("students.student_id"), nullable=False),
        sa.Column("roadmap_id", sa.Integer(), sa.ForeignKey("roadmaps.roadmap_id"), nullable=False),
        sa.Column("milestones_completed", sa.Integer(), nullable=False, server_default="0"),
        sa.Column("completion_percentage", sa.Float(), nullable=False, server_default="0"),
        sa.Column("last_reassessed_date", sa.DateTime(), nullable=False),
    )


def downgrade() -> None:
    op.drop_table("progress_tracking")
    op.drop_table("mentor_conversations")
    op.drop_table("roadmap_milestones")
    op.drop_table("roadmaps")
    op.drop_table("skill_gaps")
    op.drop_table("career_recommendations")
    op.drop_table("assessment_scores")
    op.drop_table("discovery_responses")
    op.drop_table("behavioral_metrics")
    op.drop_table("student_skills")
    op.drop_table("learning_platform_data")
    op.drop_table("academic_history")
    op.drop_table("projects")
    op.drop_table("certifications")
    op.drop_table("prompts")
    op.drop_table("discovery_questions")
    op.drop_table("skills")
    op.drop_table("students")

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\database\migrations\versions\0002_add_users_roles.py
"""add users and roles

Revision ID: 0002_add_users_roles
Revises: 0001_initial_schema
Create Date: 2026-08-01

"""
from typing import Sequence, Union

from alembic import op
import sqlalchemy as sa

revision: str = "0002_add_users_roles"
down_revision: Union[str, None] = "0001_initial_schema"
branch_labels: Union[str, Sequence[str], None] = None
depends_on: Union[str, Sequence[str], None] = None


def upgrade() -> None:
    op.create_table(
        "roles",
        sa.Column("role_id", sa.Integer(), primary_key=True, autoincrement=True),
        sa.Column("role_name", sa.String(length=50), nullable=False, unique=True),
    )

    op.create_table(
        "users",
        sa.Column("user_id", sa.Integer(), primary_key=True, autoincrement=True),
        sa.Column("email", sa.String(length=255), nullable=False, unique=True),
        sa.Column("hashed_password", sa.String(length=255), nullable=False),
        sa.Column("full_name", sa.String(length=150), nullable=True),
        sa.Column("role_id", sa.Integer(), sa.ForeignKey("roles.role_id"), nullable=False),
        sa.Column("is_active", sa.Boolean(), nullable=False, server_default=sa.true()),
        sa.Column("created_date", sa.DateTime(), nullable=False),
    )

    op.create_table(
        "token_blacklist",
        sa.Column("jti", sa.String(length=64), primary_key=True),
        sa.Column("expires_at", sa.DateTime(), nullable=False),
    )


def downgrade() -> None:
    op.drop_table("token_blacklist")
    op.drop_table("users")
    op.drop_table("roles")

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\database\migrations\versions\0003_ai_infra.py
"""AI infra: enable pgvector extension and embeddings table

Revision ID: 0003_ai_infra
Revises: 0002_add_users_roles
Create Date: 2026-08-01

"""
from typing import Sequence, Union

from alembic import op
import sqlalchemy as sa
from pgvector.sqlalchemy import Vector

revision: str = "0003_ai_infra"
down_revision: Union[str, None] = "0002_add_users_roles"
branch_labels: Union[str, Sequence[str], None] = None
depends_on: Union[str, Sequence[str], None] = None

EMBEDDING_DIM = 768


def upgrade() -> None:
    op.execute("CREATE EXTENSION IF NOT EXISTS vector")

    op.create_table(
        "embeddings",
        sa.Column("embedding_id", sa.Integer(), primary_key=True, autoincrement=True),
        sa.Column("entity_type", sa.String(length=50), nullable=False),
        sa.Column("entity_id", sa.Integer(), nullable=True),
        sa.Column("content", sa.Text(), nullable=False),
        sa.Column("vector", Vector(EMBEDDING_DIM), nullable=False),
    )
    op.execute(
        "CREATE INDEX embeddings_vector_cosine_idx ON embeddings "
        "USING ivfflat (vector vector_cosine_ops) WITH (lists = 100)"
    )


def downgrade() -> None:
    op.execute("DROP INDEX IF EXISTS embeddings_vector_cosine_idx")
    op.drop_table("embeddings")
    op.execute("DROP EXTENSION IF EXISTS vector")

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\database\migrations\versions\0004_add_user_student_link.py
"""add student_id link to users

Revision ID: 0004_add_user_student_link
Revises: 0003_ai_infra
Create Date: 2026-08-01

"""
from typing import Sequence, Union

from alembic import op
import sqlalchemy as sa

revision: str = "0004_add_user_student_link"
down_revision: Union[str, None] = "0003_ai_infra"
branch_labels: Union[str, Sequence[str], None] = None
depends_on: Union[str, Sequence[str], None] = None


def upgrade() -> None:
    op.add_column("users", sa.Column("student_id", sa.Integer(), nullable=True))
    op.create_foreign_key(
        "fk_users_student_id_students",
        "users",
        "students",
        ["student_id"],
        ["student_id"],
    )


def downgrade() -> None:
    op.drop_constraint("fk_users_student_id_students", "users", type_="foreignkey")
    op.drop_column("users", "student_id")

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\database\migrations\env.py
from logging.config import fileConfig

from alembic import context
from sqlalchemy import engine_from_config, pool

from config.settings import settings
from database.base import Base
import models  # noqa: F401  registers all ORM models on Base.metadata

config = context.config
config.set_main_option("sqlalchemy.url", settings.database_url)

if config.config_file_name is not None:
    fileConfig(config.config_file_name)

target_metadata = Base.metadata


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

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\database\migrations\script.py.mako
"""${message}

Revision ID: ${up_revision}
Revises: ${down_revision | comma,n}
Create Date: ${create_date}

"""
from typing import Sequence, Union

from alembic import op
import sqlalchemy as sa
${imports if imports else ""}

revision: str = ${repr(up_revision)}
down_revision: Union[str, None] = ${repr(down_revision)}
branch_labels: Union[str, Sequence[str], None] = ${repr(branch_labels)}
depends_on: Union[str, Sequence[str], None] = ${repr(depends_on)}


def upgrade() -> None:
    ${upgrades if upgrades else "pass"}


def downgrade() -> None:
    ${downgrades if downgrades else "pass"}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\database\__init__.py
C:\ws\agent\admission\counselor-agent\backend\database\base.py
from sqlalchemy.orm import DeclarativeBase


class Base(DeclarativeBase):
    pass

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\database\session.py
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

from config.settings import settings

# SQLite (used in tests) needs check_same_thread=False; Postgres ignores connect_args.
_connect_args = {"check_same_thread": False} if settings.database_url.startswith("sqlite") else {}
engine = create_engine(settings.database_url, pool_pre_ping=True, connect_args=_connect_args)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)


def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\middleware\__init__.py

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\models\__init__.py
from models.role import Role
from models.user import User
from models.token_blacklist import TokenBlacklist
from models.embedding import Embedding
from models.student import Student
from models.academic_history import AcademicHistory
from models.learning_platform_data import LearningPlatformData
from models.skill import Skill
from models.student_skill import StudentSkill
from models.behavioral_metric import BehavioralMetric
from models.discovery_question import DiscoveryQuestion
from models.discovery_response import DiscoveryResponse
from models.assessment_score import AssessmentScore
from models.career_recommendation import CareerRecommendation
from models.skill_gap import SkillGap
from models.roadmap import Roadmap
from models.roadmap_milestone import RoadmapMilestone
from models.certification import Certification
from models.project import Project
from models.mentor_conversation import MentorConversation
from models.progress_tracking import ProgressTracking
from models.prompt import Prompt

__all__ = [
    "Role",
    "User",
    "TokenBlacklist",
    "Embedding",
    "Student",
    "AcademicHistory",
    "LearningPlatformData",
    "Skill",
    "StudentSkill",
    "BehavioralMetric",
    "DiscoveryQuestion",
    "DiscoveryResponse",
    "AssessmentScore",
    "CareerRecommendation",
    "SkillGap",
    "Roadmap",
    "RoadmapMilestone",
    "Certification",
    "Project",
    "MentorConversation",
    "ProgressTracking",
    "Prompt",
]

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\models\academic_history.py
from datetime import datetime

from sqlalchemy import DateTime, Float, ForeignKey, Integer, String
from sqlalchemy.orm import Mapped, mapped_column

from database.base import Base


class AcademicHistory(Base):
    __tablename__ = "academic_history"

    history_id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    student_id: Mapped[int] = mapped_column(ForeignKey("students.student_id"), nullable=False)
    subject: Mapped[str] = mapped_column(String(100), nullable=False)
    score: Mapped[float] = mapped_column(Float, nullable=False)
    attendance: Mapped[float] = mapped_column(Float, nullable=True)
    learning_behavior: Mapped[str] = mapped_column(String(255), nullable=True)
    recorded_date: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\models\assessment_score.py
from datetime import datetime

from sqlalchemy import DateTime, Float, ForeignKey, Integer, String
from sqlalchemy.orm import Mapped, mapped_column

from database.base import Base


class AssessmentScore(Base):
    __tablename__ = "assessment_scores"

    score_id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    student_id: Mapped[int] = mapped_column(ForeignKey("students.student_id"), nullable=False)
    career_name: Mapped[str] = mapped_column(String(100), nullable=False)
    interest_score: Mapped[float] = mapped_column(Float, default=0)
    skill_score: Mapped[float] = mapped_column(Float, default=0)
    academic_score: Mapped[float] = mapped_column(Float, default=0)
    aptitude_score: Mapped[float] = mapped_column(Float, default=0)
    market_demand_score: Mapped[float] = mapped_column(Float, default=0)
    final_score: Mapped[float] = mapped_column(Float, default=0)
    computed_date: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\models\behavioral_metric.py
from datetime import datetime

from sqlalchemy import DateTime, Float, ForeignKey, Integer
from sqlalchemy.orm import Mapped, mapped_column

from database.base import Base


class BehavioralMetric(Base):
    __tablename__ = "behavioral_metrics"

    metric_id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    student_id: Mapped[int] = mapped_column(ForeignKey("students.student_id"), nullable=False)
    learning_speed: Mapped[float] = mapped_column(Float, nullable=True)
    consistency: Mapped[float] = mapped_column(Float, nullable=True)
    curiosity_index: Mapped[float] = mapped_column(Float, nullable=True)
    persistence: Mapped[float] = mapped_column(Float, nullable=True)
    goal_completion_rate: Mapped[float] = mapped_column(Float, nullable=True)
    recorded_date: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\models\career_recommendation.py
from datetime import datetime

from sqlalchemy import DateTime, Float, ForeignKey, Integer, String
from sqlalchemy.orm import Mapped, mapped_column

from database.base import Base


class CareerRecommendation(Base):
    __tablename__ = "career_recommendations"

    recommendation_id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    student_id: Mapped[int] = mapped_column(ForeignKey("students.student_id"), nullable=False)
    career_name: Mapped[str] = mapped_column(String(100), nullable=False)
    match_score: Mapped[float] = mapped_column(Float, nullable=False)
    rationale: Mapped[str] = mapped_column(String(1000), nullable=True)
    created_date: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\models\certification.py
from sqlalchemy import Integer, String
from sqlalchemy.orm import Mapped, mapped_column

from database.base import Base


class Certification(Base):
    __tablename__ = "certifications"

    certification_id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    career_name: Mapped[str] = mapped_column(String(100), nullable=False)
    certification_name: Mapped[str] = mapped_column(String(255), nullable=False)
    provider: Mapped[str] = mapped_column(String(100), nullable=True)

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\models\discovery_question.py
from sqlalchemy import JSON, Integer, String
from sqlalchemy.orm import Mapped, mapped_column

from database.base import Base


class DiscoveryQuestion(Base):
    __tablename__ = "discovery_questions"

    question_id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    category: Mapped[str] = mapped_column(String(50), nullable=False)
    question_text: Mapped[str] = mapped_column(String(500), nullable=False)
    options: Mapped[dict] = mapped_column(JSON, nullable=False)
    version: Mapped[int] = mapped_column(Integer, default=1)

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\models\discovery_response.py
from datetime import datetime

from sqlalchemy import DateTime, ForeignKey, Integer, String
from sqlalchemy.orm import Mapped, mapped_column

from database.base import Base


class DiscoveryResponse(Base):
    __tablename__ = "discovery_responses"

    response_id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    student_id: Mapped[int] = mapped_column(ForeignKey("students.student_id"), nullable=False)
    question_id: Mapped[int] = mapped_column(ForeignKey("discovery_questions.question_id"), nullable=False)
    selected_option: Mapped[str] = mapped_column(String(10), nullable=False)
    answered_date: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\models\embedding.py
from pgvector.sqlalchemy import Vector
from sqlalchemy import JSON, Integer, String, Text
from sqlalchemy.orm import Mapped, mapped_column

from database.base import Base

# Gemini's text-embedding-004 output dimension.
EMBEDDING_DIM = 768

# Postgres uses the real pgvector type; sqlite (used in tests) falls back to JSON.
_vector_type = Vector(EMBEDDING_DIM).with_variant(JSON(), "sqlite")


class Embedding(Base):
    __tablename__ = "embeddings"

    embedding_id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    entity_type: Mapped[str] = mapped_column(String(50), nullable=False)
    entity_id: Mapped[int] = mapped_column(Integer, nullable=True)
    content: Mapped[str] = mapped_column(Text, nullable=False)
    vector: Mapped[list[float]] = mapped_column(_vector_type, nullable=False)

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\models\learning_platform_data.py
from datetime import datetime

from sqlalchemy import DateTime, Float, ForeignKey, Integer, String
from sqlalchemy.orm import Mapped, mapped_column

from database.base import Base


class LearningPlatformData(Base):
    __tablename__ = "learning_platform_data"

    record_id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    student_id: Mapped[int] = mapped_column(ForeignKey("students.student_id"), nullable=False)
    platform: Mapped[str] = mapped_column(String(100), nullable=False)
    courses_completed: Mapped[int] = mapped_column(Integer, default=0)
    hours_spent: Mapped[float] = mapped_column(Float, default=0)
    skill_area: Mapped[str] = mapped_column(String(100), nullable=True)
    recorded_date: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\models\mentor_conversation.py
from datetime import datetime

from sqlalchemy import DateTime, ForeignKey, Integer, String
from sqlalchemy.orm import Mapped, mapped_column

from database.base import Base


class MentorConversation(Base):
    __tablename__ = "mentor_conversations"

    conversation_id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    student_id: Mapped[int] = mapped_column(ForeignKey("students.student_id"), nullable=False)
    message: Mapped[str] = mapped_column(String(2000), nullable=False)
    reply: Mapped[str] = mapped_column(String(2000), nullable=True)
    related_career: Mapped[str] = mapped_column(String(100), nullable=True)
    created_date: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\models\progress_tracking.py
from datetime import datetime

from sqlalchemy import DateTime, Float, ForeignKey, Integer
from sqlalchemy.orm import Mapped, mapped_column

from database.base import Base


class ProgressTracking(Base):
    __tablename__ = "progress_tracking"

    tracking_id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    student_id: Mapped[int] = mapped_column(ForeignKey("students.student_id"), nullable=False)
    roadmap_id: Mapped[int] = mapped_column(ForeignKey("roadmaps.roadmap_id"), nullable=False)
    milestones_completed: Mapped[int] = mapped_column(Integer, default=0)
    completion_percentage: Mapped[float] = mapped_column(Float, default=0)
    last_reassessed_date: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\models\project.py
from sqlalchemy import Integer, String
from sqlalchemy.orm import Mapped, mapped_column

from database.base import Base


class Project(Base):
    __tablename__ = "projects"

    project_id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    career_name: Mapped[str] = mapped_column(String(100), nullable=False)
    project_name: Mapped[str] = mapped_column(String(255), nullable=False)
    description: Mapped[str] = mapped_column(String(1000), nullable=True)
    difficulty_level: Mapped[str] = mapped_column(String(20), nullable=True)

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\models\prompt.py
from sqlalchemy import Integer, String, Text
from sqlalchemy.orm import Mapped, mapped_column

from database.base import Base


class Prompt(Base):
    __tablename__ = "prompts"

    prompt_id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    agent_name: Mapped[str] = mapped_column(String(100), nullable=False)
    prompt_type: Mapped[str] = mapped_column(String(100), nullable=False)
    prompt_text: Mapped[str] = mapped_column(Text, nullable=False)
    version: Mapped[int] = mapped_column(Integer, default=1)

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\models\roadmap_milestone.py
from datetime import datetime

from sqlalchemy import JSON, DateTime, ForeignKey, Integer, String
from sqlalchemy.orm import Mapped, mapped_column

from database.base import Base


class RoadmapMilestone(Base):
    __tablename__ = "roadmap_milestones"

    milestone_id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    roadmap_id: Mapped[int] = mapped_column(ForeignKey("roadmaps.roadmap_id"), nullable=False)
    month_range: Mapped[str] = mapped_column(String(20), nullable=False)
    topics: Mapped[list] = mapped_column(JSON, nullable=False)
    resources: Mapped[list] = mapped_column(JSON, nullable=True)
    project: Mapped[str] = mapped_column(String(255), nullable=True)
    status: Mapped[str] = mapped_column(String(20), default="PENDING")
    completed_date: Mapped[datetime | None] = mapped_column(DateTime, nullable=True)

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\models\roadmap.py
from datetime import datetime

from sqlalchemy import DateTime, ForeignKey, Integer, String
from sqlalchemy.orm import Mapped, mapped_column

from database.base import Base


class Roadmap(Base):
    __tablename__ = "roadmaps"

    roadmap_id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    student_id: Mapped[int] = mapped_column(ForeignKey("students.student_id"), nullable=False)
    recommendation_id: Mapped[int] = mapped_column(
        ForeignKey("career_recommendations.recommendation_id"), nullable=False
    )
    goal: Mapped[str] = mapped_column(String(150), nullable=False)
    duration_months: Mapped[int] = mapped_column(Integer, nullable=False)
    status: Mapped[str] = mapped_column(String(20), default="ACTIVE")
    created_date: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\models\roadmap.py
from datetime import datetime

from sqlalchemy import DateTime, ForeignKey, Integer, String
from sqlalchemy.orm import Mapped, mapped_column

from database.base import Base


class Roadmap(Base):
    __tablename__ = "roadmaps"

    roadmap_id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    student_id: Mapped[int] = mapped_column(ForeignKey("students.student_id"), nullable=False)
    recommendation_id: Mapped[int] = mapped_column(
        ForeignKey("career_recommendations.recommendation_id"), nullable=False
    )
    goal: Mapped[str] = mapped_column(String(150), nullable=False)
    duration_months: Mapped[int] = mapped_column(Integer, nullable=False)
    status: Mapped[str] = mapped_column(String(20), default="ACTIVE")
    created_date: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\models\role.py
from sqlalchemy import Integer, String
from sqlalchemy.orm import Mapped, mapped_column

from database.base import Base


class Role(Base):
    __tablename__ = "roles"

    role_id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    role_name: Mapped[str] = mapped_column(String(50), nullable=False, unique=True)

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\models\skill_gap.py
from sqlalchemy import ForeignKey, Integer, String
from sqlalchemy.orm import Mapped, mapped_column

from database.base import Base


class SkillGap(Base):
    __tablename__ = "skill_gaps"

    gap_id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    recommendation_id: Mapped[int] = mapped_column(
        ForeignKey("career_recommendations.recommendation_id"), nullable=False
    )
    skill_id: Mapped[int] = mapped_column(ForeignKey("skills.skill_id"), nullable=False)
    status: Mapped[str] = mapped_column(String(10), nullable=False)  # MATCHED / MISSING

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\models\skill.py
from sqlalchemy import Integer, String
from sqlalchemy.orm import Mapped, mapped_column

from database.base import Base


class Skill(Base):
    __tablename__ = "skills"

    skill_id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    skill_name: Mapped[str] = mapped_column(String(100), nullable=False, unique=True)
    skill_type: Mapped[str] = mapped_column(String(20), nullable=False)  # TECHNICAL / SOFT

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\models\student_skill.py
from datetime import datetime

from sqlalchemy import DateTime, Float, ForeignKey, Integer
from sqlalchemy.orm import Mapped, mapped_column

from database.base import Base


class StudentSkill(Base):
    __tablename__ = "student_skills"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    student_id: Mapped[int] = mapped_column(ForeignKey("students.student_id"), nullable=False)
    skill_id: Mapped[int] = mapped_column(ForeignKey("skills.skill_id"), nullable=False)
    proficiency_score: Mapped[float] = mapped_column(Float, nullable=False)
    assessed_date: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\models\student.py
from datetime import datetime

from sqlalchemy import JSON, DateTime, Integer, String
from sqlalchemy.orm import Mapped, mapped_column

from database.base import Base


class Student(Base):
    __tablename__ = "students"

    student_id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    name: Mapped[str] = mapped_column(String(150), nullable=False)
    age: Mapped[int] = mapped_column(Integer, nullable=False)
    education: Mapped[str] = mapped_column(String(150), nullable=False)
    location: Mapped[str] = mapped_column(String(150), nullable=True)
    board: Mapped[str] = mapped_column(String(50), nullable=True)
    subjects: Mapped[list] = mapped_column(JSON, nullable=True)
    created_date: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\models\token_blacklist.py
from datetime import datetime

from sqlalchemy import DateTime, String
from sqlalchemy.orm import Mapped, mapped_column

from database.base import Base


class TokenBlacklist(Base):
    """Revoked JWT ids (logout / used password-reset tokens)."""

    __tablename__ = "token_blacklist"

    jti: Mapped[str] = mapped_column(String(64), primary_key=True)
    expires_at: Mapped[datetime] = mapped_column(DateTime, nullable=False)

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\models\user.py
from datetime import datetime

from sqlalchemy import Boolean, DateTime, ForeignKey, Integer, String
from sqlalchemy.orm import Mapped, mapped_column, relationship

from database.base import Base
from models.role import Role


class User(Base):
    __tablename__ = "users"

    user_id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    email: Mapped[str] = mapped_column(String(255), nullable=False, unique=True)
    hashed_password: Mapped[str] = mapped_column(String(255), nullable=False)
    full_name: Mapped[str] = mapped_column(String(150), nullable=True)
    role_id: Mapped[int] = mapped_column(ForeignKey("roles.role_id"), nullable=False)
    # Links a STUDENT-role login to their own record; null for non-student roles.
    student_id: Mapped[int | None] = mapped_column(ForeignKey("students.student_id"), nullable=True)
    is_active: Mapped[bool] = mapped_column(Boolean, default=True)
    created_date: Mapped[datetime] = mapped_column(DateTime, default=datetime.utcnow)

    role: Mapped[Role] = relationship(Role, lazy="joined")

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\prompts\__init__.py
C:\ws\agent\admission\counselor-agent\backend\prompts\prompt_loader.py
from sqlalchemy.orm import Session

from repositories.prompt_repository import get_latest


class PromptNotFoundError(Exception):
    pass


def load_prompt(db: Session, agent_name: str, prompt_type: str) -> str:
    """Loads the latest stored prompt version for use by any agent node."""
    prompt = get_latest(db, agent_name, prompt_type)
    if not prompt:
        raise PromptNotFoundError(f"No prompt found for agent={agent_name!r}, type={prompt_type!r}")
    return prompt.prompt_text

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\repositories\__init__.py
C:\ws\agent\admission\counselor-agent\backend\repositories\academic_history_repository.py
from sqlalchemy.orm import Session

from models.academic_history import AcademicHistory


def create_academic_history(db: Session, student_id: int, data: dict) -> AcademicHistory:
    record = AcademicHistory(student_id=student_id, **data)
    db.add(record)
    db.commit()
    db.refresh(record)
    return record


def list_academic_history(db: Session, student_id: int) -> list[AcademicHistory]:
    return (
        db.query(AcademicHistory)
        .filter(AcademicHistory.student_id == student_id)
        .order_by(AcademicHistory.recorded_date)
        .all()
    )

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\repositories\assessment_repository.py
from sqlalchemy.orm import Session

from models.assessment_score import AssessmentScore


def save_assessment_score(db: Session, student_id: int, career_name: str, scores: dict) -> AssessmentScore:
    record = AssessmentScore(student_id=student_id, career_name=career_name, **scores)
    db.add(record)
    db.commit()
    db.refresh(record)
    return record


def list_assessment_scores(db: Session, student_id: int) -> list[AssessmentScore]:
    return (
        db.query(AssessmentScore)
        .filter(AssessmentScore.student_id == student_id)
        .order_by(AssessmentScore.computed_date)
        .all()
    )

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\repositories\catalog_repository.py
from sqlalchemy.orm import Session

from models.certification import Certification
from models.project import Project


def list_certifications(db: Session) -> list[Certification]:
    return db.query(Certification).order_by(Certification.certification_id).all()


def get_certification(db: Session, certification_id: int) -> Certification | None:
    return db.query(Certification).filter(Certification.certification_id == certification_id).first()


def create_certification(db: Session, data: dict) -> Certification:
    certification = Certification(**data)
    db.add(certification)
    db.commit()
    db.refresh(certification)
    return certification


def update_certification(db: Session, certification: Certification, data: dict) -> Certification:
    for key, value in data.items():
        setattr(certification, key, value)
    db.commit()
    db.refresh(certification)
    return certification


def delete_certification(db: Session, certification: Certification) -> None:
    db.delete(certification)
    db.commit()


def list_projects(db: Session) -> list[Project]:
    return db.query(Project).order_by(Project.project_id).all()


def get_project(db: Session, project_id: int) -> Project | None:
    return db.query(Project).filter(Project.project_id == project_id).first()


def create_project(db: Session, data: dict) -> Project:
    project = Project(**data)
    db.add(project)
    db.commit()
    db.refresh(project)
    return project


def update_project(db: Session, project: Project, data: dict) -> Project:
    for key, value in data.items():
        setattr(project, key, value)
    db.commit()
    db.refresh(project)
    return project


def delete_project(db: Session, project: Project) -> None:
    db.delete(project)
    db.commit()

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\repositories\discovery_repository.py
from sqlalchemy.orm import Session

from models.discovery_question import DiscoveryQuestion
from models.discovery_response import DiscoveryResponse


def get_question(db: Session, question_id: int) -> DiscoveryQuestion | None:
    return db.query(DiscoveryQuestion).filter(DiscoveryQuestion.question_id == question_id).first()


def get_next_question(db: Session, student_id: int) -> DiscoveryQuestion | None:
    """Returns the next unanswered question for the student, cycling through categories in insertion order."""
    answered_ids = [
        row[0]
        for row in db.query(DiscoveryResponse.question_id).filter(DiscoveryResponse.student_id == student_id).all()
    ]
    query = db.query(DiscoveryQuestion)
    if answered_ids:
        query = query.filter(~DiscoveryQuestion.question_id.in_(answered_ids))
    return query.order_by(DiscoveryQuestion.question_id).first()


def record_answer(db: Session, student_id: int, question_id: int, selected_option: str) -> DiscoveryResponse:
    response = DiscoveryResponse(student_id=student_id, question_id=question_id, selected_option=selected_option)
    db.add(response)
    db.commit()
    db.refresh(response)
    return response


def list_responses(db: Session, student_id: int) -> list[DiscoveryResponse]:
    return (
        db.query(DiscoveryResponse)
        .filter(DiscoveryResponse.student_id == student_id)
        .order_by(DiscoveryResponse.answered_date)
        .all()
    )


def list_questions(db: Session) -> list[DiscoveryQuestion]:
    return db.query(DiscoveryQuestion).order_by(DiscoveryQuestion.question_id).all()


def create_question(db: Session, category: str, question_text: str, options: dict) -> DiscoveryQuestion:
    question = DiscoveryQuestion(category=category, question_text=question_text, options=options, version=1)
    db.add(question)
    db.commit()
    db.refresh(question)
    return question


def update_question(db: Session, question: DiscoveryQuestion, data: dict) -> DiscoveryQuestion:
    for key, value in data.items():
        setattr(question, key, value)
    # Any edit bumps the version so consumers can tell the question text/options changed.
    question.version += 1
    db.commit()
    db.refresh(question)
    return question

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\repositories\learning_platform_repository.py
from sqlalchemy.orm import Session

from models.learning_platform_data import LearningPlatformData


def create_learning_record(db: Session, student_id: int, data: dict) -> LearningPlatformData:
    record = LearningPlatformData(student_id=student_id, **data)
    db.add(record)
    db.commit()
    db.refresh(record)
    return record


def list_learning_records(db: Session, student_id: int) -> list[LearningPlatformData]:
    return (
        db.query(LearningPlatformData)
        .filter(LearningPlatformData.student_id == student_id)
        .order_by(LearningPlatformData.recorded_date)
        .all()
    )

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\repositories\mentor_repository.py
from sqlalchemy.orm import Session

from models.mentor_conversation import MentorConversation


def create_conversation(
    db: Session, student_id: int, message: str, reply: str, related_career: str | None
) -> MentorConversation:
    record = MentorConversation(
        student_id=student_id, message=message, reply=reply, related_career=related_career
    )
    db.add(record)
    db.commit()
    db.refresh(record)
    return record


def list_conversations(db: Session, student_id: int) -> list[MentorConversation]:
    return (
        db.query(MentorConversation)
        .filter(MentorConversation.student_id == student_id)
        .order_by(MentorConversation.created_date)
        .all()
    )

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\repositories\progress_repository.py
from datetime import datetime

from sqlalchemy.orm import Session

from models.progress_tracking import ProgressTracking


def get_progress_by_roadmap(db: Session, roadmap_id: int) -> ProgressTracking | None:
    return db.query(ProgressTracking).filter(ProgressTracking.roadmap_id == roadmap_id).first()


def create_progress(db: Session, student_id: int, roadmap_id: int) -> ProgressTracking:
    progress = ProgressTracking(student_id=student_id, roadmap_id=roadmap_id)
    db.add(progress)
    db.commit()
    db.refresh(progress)
    return progress


def update_progress(
    db: Session,
    progress: ProgressTracking,
    milestones_completed: int,
    completion_percentage: float,
    last_reassessed_date: datetime,
) -> ProgressTracking:
    progress.milestones_completed = milestones_completed
    progress.completion_percentage = completion_percentage
    progress.last_reassessed_date = last_reassessed_date
    db.commit()
    db.refresh(progress)
    return progress


def list_progress_by_student(db: Session, student_id: int) -> list[ProgressTracking]:
    return (
        db.query(ProgressTracking)
        .filter(ProgressTracking.student_id == student_id)
        .order_by(ProgressTracking.tracking_id)
        .all()
    )

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\repositories\prompt_repository.py
from sqlalchemy.orm import Session

from models.prompt import Prompt


def get_latest(db: Session, agent_name: str, prompt_type: str) -> Prompt | None:
    return (
        db.query(Prompt)
        .filter(Prompt.agent_name == agent_name, Prompt.prompt_type == prompt_type)
        .order_by(Prompt.version.desc())
        .first()
    )


def create_prompt(db: Session, agent_name: str, prompt_type: str, prompt_text: str) -> Prompt:
    """Adds a new immutable version for (agent_name, prompt_type); never overwrites history."""
    latest = get_latest(db, agent_name, prompt_type)
    version = latest.version + 1 if latest else 1
    prompt = Prompt(agent_name=agent_name, prompt_type=prompt_type, prompt_text=prompt_text, version=version)
    db.add(prompt)
    db.commit()
    db.refresh(prompt)
    return prompt


def get_prompt(db: Session, prompt_id: int) -> Prompt | None:
    return db.query(Prompt).filter(Prompt.prompt_id == prompt_id).first()


def list_prompts(db: Session, agent_name: str | None = None) -> list[Prompt]:
    query = db.query(Prompt)
    if agent_name:
        query = query.filter(Prompt.agent_name == agent_name)
    return query.order_by(Prompt.agent_name, Prompt.prompt_type, Prompt.version.desc()).all()


def delete_prompt(db: Session, prompt: Prompt) -> None:
    db.delete(prompt)
    db.commit()

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\repositories\recommendation_repository.py
from sqlalchemy.orm import Session

from models.career_recommendation import CareerRecommendation


def create_recommendation(
    db: Session, student_id: int, career_name: str, match_score: float, rationale: str
) -> CareerRecommendation:
    record = CareerRecommendation(
        student_id=student_id, career_name=career_name, match_score=match_score, rationale=rationale
    )
    db.add(record)
    db.commit()
    db.refresh(record)
    return record


def list_recommendations(db: Session, student_id: int) -> list[CareerRecommendation]:
    return (
        db.query(CareerRecommendation)
        .filter(CareerRecommendation.student_id == student_id)
        .order_by(CareerRecommendation.created_date)
        .all()
    )


def get_recommendation(db: Session, recommendation_id: int) -> CareerRecommendation | None:
    return (
        db.query(CareerRecommendation)
        .filter(CareerRecommendation.recommendation_id == recommendation_id)
        .first()
    )

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\repositories\roadmap_repository.py
from sqlalchemy.orm import Session

from models.certification import Certification
from models.project import Project
from models.roadmap import Roadmap
from models.roadmap_milestone import RoadmapMilestone


def create_roadmap(db: Session, student_id: int, recommendation_id: int, goal: str, duration_months: int) -> Roadmap:
    roadmap = Roadmap(
        student_id=student_id, recommendation_id=recommendation_id, goal=goal, duration_months=duration_months
    )
    db.add(roadmap)
    db.commit()
    db.refresh(roadmap)
    return roadmap


def create_milestone(
    db: Session,
    roadmap_id: int,
    month_range: str,
    topics: list[str],
    resources: list[str],
    project: str | None,
) -> RoadmapMilestone:
    milestone = RoadmapMilestone(
        roadmap_id=roadmap_id, month_range=month_range, topics=topics, resources=resources, project=project
    )
    db.add(milestone)
    db.commit()
    db.refresh(milestone)
    return milestone


def get_roadmap(db: Session, roadmap_id: int) -> Roadmap | None:
    return db.query(Roadmap).filter(Roadmap.roadmap_id == roadmap_id).first()


def list_roadmaps(db: Session, student_id: int) -> list[Roadmap]:
    return db.query(Roadmap).filter(Roadmap.student_id == student_id).order_by(Roadmap.created_date).all()


def list_milestones(db: Session, roadmap_id: int) -> list[RoadmapMilestone]:
    return (
        db.query(RoadmapMilestone)
        .filter(RoadmapMilestone.roadmap_id == roadmap_id)
        .order_by(RoadmapMilestone.milestone_id)
        .all()
    )


def get_milestone(db: Session, milestone_id: int) -> RoadmapMilestone | None:
    return db.query(RoadmapMilestone).filter(RoadmapMilestone.milestone_id == milestone_id).first()


def list_certifications_by_career(db: Session, career_name: str) -> list[Certification]:
    return db.query(Certification).filter(Certification.career_name == career_name).all()


def list_projects_by_career(db: Session, career_name: str) -> list[Project]:
    return db.query(Project).filter(Project.career_name == career_name).all()

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\repositories\skill_gap_repository.py
from sqlalchemy.orm import Session

from models.skill import Skill
from models.skill_gap import SkillGap


def delete_gaps_for_recommendation(db: Session, recommendation_id: int) -> None:
    db.query(SkillGap).filter(SkillGap.recommendation_id == recommendation_id).delete()
    db.commit()


def create_skill_gap(db: Session, recommendation_id: int, skill_id: int, status: str) -> SkillGap:
    record = SkillGap(recommendation_id=recommendation_id, skill_id=skill_id, status=status)
    db.add(record)
    db.commit()
    db.refresh(record)
    return record


def list_skill_gaps(db: Session, recommendation_id: int) -> list[tuple[SkillGap, str]]:
    return (
        db.query(SkillGap, Skill.skill_name)
        .join(Skill, SkillGap.skill_id == Skill.skill_id)
        .filter(SkillGap.recommendation_id == recommendation_id)
        .all()
    )

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\repositories\skill_repository.py
from sqlalchemy.orm import Session

from models.behavioral_metric import BehavioralMetric
from models.skill import Skill
from models.student_skill import StudentSkill


def get_or_create_skill(db: Session, skill_name: str, skill_type: str) -> Skill:
    skill = db.query(Skill).filter(Skill.skill_name == skill_name).first()
    if skill:
        return skill
    skill = Skill(skill_name=skill_name, skill_type=skill_type)
    db.add(skill)
    db.commit()
    db.refresh(skill)
    return skill


def record_student_skill(db: Session, student_id: int, skill_id: int, proficiency_score: float) -> StudentSkill:
    record = StudentSkill(student_id=student_id, skill_id=skill_id, proficiency_score=proficiency_score)
    db.add(record)
    db.commit()
    db.refresh(record)
    return record


def list_student_skills(db: Session, student_id: int) -> list[tuple[StudentSkill, Skill]]:
    return (
        db.query(StudentSkill, Skill)
        .join(Skill, StudentSkill.skill_id == Skill.skill_id)
        .filter(StudentSkill.student_id == student_id)
        .order_by(StudentSkill.assessed_date)
        .all()
    )


def record_behavioral_metric(db: Session, student_id: int, data: dict) -> BehavioralMetric:
    record = BehavioralMetric(student_id=student_id, **data)
    db.add(record)
    db.commit()
    db.refresh(record)
    return record


def list_behavioral_metrics(db: Session, student_id: int) -> list[BehavioralMetric]:
    return (
        db.query(BehavioralMetric)
        .filter(BehavioralMetric.student_id == student_id)
        .order_by(BehavioralMetric.recorded_date)
        .all()
    )

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\repositories\student_repository.py
from sqlalchemy.orm import Session

from models.student import Student


def create_student(db: Session, data: dict) -> Student:
    student = Student(**data)
    db.add(student)
    db.commit()
    db.refresh(student)
    return student


def get_student(db: Session, student_id: int) -> Student | None:
    return db.query(Student).filter(Student.student_id == student_id).first()


def list_students(db: Session, skip: int = 0, limit: int = 50) -> list[Student]:
    return db.query(Student).order_by(Student.student_id).offset(skip).limit(limit).all()


def update_student(db: Session, student: Student, data: dict) -> Student:
    for key, value in data.items():
        setattr(student, key, value)
    db.commit()
    db.refresh(student)
    return student


def delete_student(db: Session, student: Student) -> None:
    db.delete(student)
    db.commit()

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\repositories\user_repository.py
from sqlalchemy.orm import Session

from models.role import Role
from models.user import User


def get_user_by_email(db: Session, email: str) -> User | None:
    return db.query(User).filter(User.email == email).first()


def get_user_by_id(db: Session, user_id: int) -> User | None:
    return db.query(User).filter(User.user_id == user_id).first()


def get_role_by_name(db: Session, role_name: str) -> Role | None:
    return db.query(Role).filter(Role.role_name == role_name).first()


def create_user(
    db: Session,
    email: str,
    hashed_password: str,
    full_name: str | None,
    role_id: int,
    student_id: int | None = None,
) -> User:
    user = User(
        email=email,
        hashed_password=hashed_password,
        full_name=full_name,
        role_id=role_id,
        student_id=student_id,
    )
    db.add(user)
    db.commit()
    db.refresh(user)
    return user


def update_password(db: Session, user: User, hashed_password: str) -> None:
    user.hashed_password = hashed_password
    db.commit()

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\schemas\__init__.py
C:\ws\agent\admission\counselor-agent\backend\schemas\academic_history.py
from datetime import datetime

from pydantic import BaseModel, Field


class AcademicHistoryCreate(BaseModel):
    subject: str = Field(min_length=1, max_length=100)
    score: float = Field(ge=0, le=100)
    attendance: float | None = Field(default=None, ge=0, le=100)
    learning_behavior: str | None = Field(default=None, max_length=255)


class AcademicHistoryOut(AcademicHistoryCreate):
    history_id: int
    student_id: int
    recorded_date: datetime

    model_config = {"from_attributes": True}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\schemas\analytics.py
from pydantic import BaseModel


class CareerDemandPoint(BaseModel):
    career_name: str
    market_demand_score: float


class CareerDemandTrendsOut(BaseModel):
    chart_type: str
    data: list[CareerDemandPoint]


class SkillGapHeatmapEntry(BaseModel):
    career_name: str
    skill_gap_counts: dict[str, int]


class SkillGapHeatmapOut(BaseModel):
    chart_type: str
    data: list[SkillGapHeatmapEntry]


class CareerCompletionRate(BaseModel):
    career_name: str
    average_completion_percentage: float


class RoadmapCompletionRateOut(BaseModel):
    chart_type: str
    overall_average_completion_percentage: float
    by_career: list[CareerCompletionRate]


class TopRecommendedCareer(BaseModel):
    career_name: str
    recommendation_count: int


class TopRecommendedCareersOut(BaseModel):
    chart_type: str
    data: list[TopRecommendedCareer]


class FunnelStage(BaseModel):
    stage: str
    count: int


class CertificationUptakeOut(BaseModel):
    chart_type: str
    data: list[FunnelStage]


class DashboardSummaryOut(BaseModel):
    career_demand_trends: CareerDemandTrendsOut
    cohort_skill_gap_heatmap: SkillGapHeatmapOut
    roadmap_completion_rate: RoadmapCompletionRateOut
    top_recommended_careers: TopRecommendedCareersOut
    certification_uptake: CertificationUptakeOut
    active_students: int
    assessments_completed_today: int

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\schemas\assessment.py
from datetime import datetime

from pydantic import BaseModel


class AssessmentRequest(BaseModel):
    student_id: int
    career_names: list[str] | None = None


class AssessmentScoreOut(BaseModel):
    score_id: int
    student_id: int
    career_name: str
    interest_score: float
    skill_score: float
    academic_score: float
    aptitude_score: float
    market_demand_score: float
    final_score: float
    computed_date: datetime

    model_config = {"from_attributes": True}


class AssessmentResponse(BaseModel):
    scores: dict[str, float]
    details: list[AssessmentScoreOut]

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\schemas\auth.py
from pydantic import BaseModel, EmailStr, Field


class LoginRequest(BaseModel):
    email: EmailStr
    password: str


class TokenResponse(BaseModel):
    access_token: str
    refresh_token: str
    token_type: str = "bearer"


class RefreshRequest(BaseModel):
    refresh_token: str


class AccessTokenResponse(BaseModel):
    access_token: str
    token_type: str = "bearer"


class PasswordResetRequest(BaseModel):
    email: EmailStr


class PasswordResetRequestResponse(BaseModel):
    # Returned directly since no email delivery service is wired up yet.
    reset_token: str


class PasswordResetConfirmRequest(BaseModel):
    reset_token: str
    new_password: str = Field(min_length=8)


class UserOut(BaseModel):
    user_id: int
    email: EmailStr
    full_name: str | None
    role: str
    is_active: bool
    student_id: int | None = None

    model_config = {"from_attributes": True}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\schemas\catalog.py
from pydantic import BaseModel, Field


class CertificationCreate(BaseModel):
    career_name: str = Field(min_length=1, max_length=100)
    certification_name: str = Field(min_length=1, max_length=255)
    provider: str | None = Field(default=None, max_length=100)


class CertificationUpdate(BaseModel):
    career_name: str | None = Field(default=None, min_length=1, max_length=100)
    certification_name: str | None = Field(default=None, min_length=1, max_length=255)
    provider: str | None = Field(default=None, max_length=100)


class CertificationOut(BaseModel):
    certification_id: int
    career_name: str
    certification_name: str
    provider: str | None

    model_config = {"from_attributes": True}


class ProjectCreate(BaseModel):
    career_name: str = Field(min_length=1, max_length=100)
    project_name: str = Field(min_length=1, max_length=255)
    description: str | None = Field(default=None, max_length=1000)
    difficulty_level: str | None = Field(default=None, max_length=20)


class ProjectUpdate(BaseModel):
    career_name: str | None = Field(default=None, min_length=1, max_length=100)
    project_name: str | None = Field(default=None, min_length=1, max_length=255)
    description: str | None = Field(default=None, max_length=1000)
    difficulty_level: str | None = Field(default=None, max_length=20)


class ProjectOut(BaseModel):
    project_id: int
    career_name: str
    project_name: str
    description: str | None
    difficulty_level: str | None

    model_config = {"from_attributes": True}


class CareerDefinitionOut(BaseModel):
    career_name: str
    required_skills: list[str]
    target_skills: list[str]
    interest_map: dict[str, str]
    academic_subjects: list[str]
    market_demand_score: float

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\schemas\discovery.py
from pydantic import BaseModel, Field


class DiscoveryQuestionOut(BaseModel):
    question_id: int
    category: str
    question_text: str
    options: dict
    version: int

    model_config = {"from_attributes": True}


class DiscoveryAnswerRequest(BaseModel):
    student_id: int
    question_id: int
    selected_option: str = Field(min_length=1, max_length=10)


class DiscoveryResponseOut(BaseModel):
    response_id: int
    student_id: int
    question_id: int
    selected_option: str

    model_config = {"from_attributes": True}


class DiscoveryQuestionCreate(BaseModel):
    category: str = Field(min_length=1, max_length=50)
    question_text: str = Field(min_length=1, max_length=500)
    options: dict[str, str] = Field(min_length=2)


class DiscoveryQuestionUpdate(BaseModel):
    category: str | None = Field(default=None, min_length=1, max_length=50)
    question_text: str | None = Field(default=None, min_length=1, max_length=500)
    options: dict[str, str] | None = Field(default=None, min_length=2)

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\schemas\gap_analysis.py
from pydantic import BaseModel


class GapAnalysisRequest(BaseModel):
    recommendation_id: int


class SkillGapOut(BaseModel):
    gap_id: int
    skill_id: int
    skill_name: str
    status: str


class GapAnalysisResponse(BaseModel):
    recommendation_id: int
    matched: list[str]
    missing: list[str]
    details: list[SkillGapOut]

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\schemas\learning_platform_data.py
from datetime import datetime

from pydantic import BaseModel, Field


class LearningPlatformDataCreate(BaseModel):
    platform: str = Field(min_length=1, max_length=100)
    courses_completed: int = Field(default=0, ge=0)
    hours_spent: float = Field(default=0, ge=0)
    skill_area: str | None = Field(default=None, max_length=100)


class LearningPlatformDataOut(LearningPlatformDataCreate):
    record_id: int
    student_id: int
    recorded_date: datetime

    model_config = {"from_attributes": True}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\schemas\mentor.py
from datetime import datetime

from pydantic import BaseModel, Field


class MentorChatRequest(BaseModel):
    student_id: int = Field(alias="studentId")
    message: str = Field(min_length=1, max_length=2000)

    model_config = {"populate_by_name": True}


class MentorChatResponse(BaseModel):
    reply: str
    related_career: str | None = Field(default=None, alias="relatedCareer")

    model_config = {"populate_by_name": True}


class MentorConversationOut(BaseModel):
    conversation_id: int
    student_id: int
    message: str
    reply: str | None = None
    related_career: str | None = None
    created_date: datetime

    model_config = {"from_attributes": True}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\schemas\orchestrator.py
from pydantic import BaseModel


class CounselorFlowRequest(BaseModel):
    student_id: int
    career_names: list[str] | None = None
    mentor_message: str | None = None


class CounselorFlowResponse(BaseModel):
    profile: dict | None = None
    profile_error: str | None = None
    learning_history: dict | None = None
    learning_error: str | None = None
    assessment_scores: list[dict] | None = None
    assessment_error: str | None = None
    recommendations: list[dict] | None = None
    top_recommendation_id: int | None = None
    recommendation_error: str | None = None
    roadmap: dict | None = None
    roadmap_error: str | None = None
    mentor_reply: dict | None = None
    mentor_error: str | None = None
    progress: list[dict] | None = None
    progress_error: str | None = None

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\schemas\progress.py
from datetime import datetime

from pydantic import BaseModel


class ProgressMilestoneCompleteRequest(BaseModel):
    milestone_id: int


class UpdatedRecommendationOut(BaseModel):
    career_name: str
    match_score: float

    model_config = {"from_attributes": True}


class ProgressOut(BaseModel):
    tracking_id: int
    student_id: int
    roadmap_id: int
    milestones_completed: int
    total_milestones: int
    completion_percentage: float
    last_reassessed_date: datetime


class ProgressCompleteResponse(ProgressOut):
    reassessment_triggered: bool
    updated_recommendation: UpdatedRecommendationOut | None = None

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\schemas\prompt.py
from pydantic import BaseModel, Field


class PromptBase(BaseModel):
    agent_name: str = Field(min_length=1, max_length=100)
    prompt_type: str = Field(min_length=1, max_length=100)


class PromptCreate(PromptBase):
    prompt_text: str = Field(min_length=1)


class PromptOut(PromptBase):
    prompt_id: int
    prompt_text: str
    version: int

    model_config = {"from_attributes": True}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\schemas\recommendation.py
from datetime import datetime

from pydantic import BaseModel


class RecommendationRequest(BaseModel):
    student_id: int
    career_names: list[str] | None = None
    top_n: int | None = None


class RecommendationOut(BaseModel):
    recommendation_id: int
    student_id: int
    career_name: str
    match_score: float
    rationale: str | None = None
    created_date: datetime

    model_config = {"from_attributes": True}


class RecommendationResponse(BaseModel):
    recommendations: list[RecommendationOut]

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\schemas\roadmap.py
from datetime import datetime

from pydantic import BaseModel


class RoadmapGenerateRequest(BaseModel):
    recommendation_id: int
    duration_months: int = 12


class RoadmapMilestoneOut(BaseModel):
    milestone_id: int
    roadmap_id: int
    month_range: str
    topics: list[str]
    resources: list[str] | None = None
    project: str | None = None
    status: str
    completed_date: datetime | None = None

    model_config = {"from_attributes": True}


class RoadmapOut(BaseModel):
    roadmap_id: int
    student_id: int
    recommendation_id: int
    goal: str
    duration_months: int
    status: str
    created_date: datetime
    milestones: list[RoadmapMilestoneOut]

    model_config = {"from_attributes": True}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\schemas\skill_assessment.py
from datetime import datetime

from pydantic import BaseModel, Field


class SkillAssessmentItem(BaseModel):
    skill_name: str = Field(min_length=1, max_length=100)
    skill_type: str = Field(pattern="^(TECHNICAL|SOFT)$")
    proficiency_score: float = Field(ge=0, le=100)


class BehavioralMetricCreate(BaseModel):
    learning_speed: float | None = Field(default=None, ge=0, le=100)
    consistency: float | None = Field(default=None, ge=0, le=100)
    curiosity_index: float | None = Field(default=None, ge=0, le=100)
    persistence: float | None = Field(default=None, ge=0, le=100)
    goal_completion_rate: float | None = Field(default=None, ge=0, le=100)


class SkillAssessmentRequest(BaseModel):
    skills: list[SkillAssessmentItem] = Field(default_factory=list)
    behavioral_metrics: BehavioralMetricCreate | None = None


class StudentSkillOut(BaseModel):
    id: int
    student_id: int
    skill_id: int
    skill_name: str
    skill_type: str
    proficiency_score: float
    assessed_date: datetime

    model_config = {"from_attributes": True}


class BehavioralMetricOut(BehavioralMetricCreate):
    metric_id: int
    student_id: int
    recorded_date: datetime

    model_config = {"from_attributes": True}


class SkillAssessmentOut(BaseModel):
    skills: list[StudentSkillOut]
    behavioral_metric: BehavioralMetricOut | None = None

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\schemas\student_history.py
from pydantic import BaseModel

from schemas.academic_history import AcademicHistoryOut
from schemas.learning_platform_data import LearningPlatformDataOut


class StudentHistoryOut(BaseModel):
    academic_history: list[AcademicHistoryOut]
    learning_platform_data: list[LearningPlatformDataOut]

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\schemas\student.py
from datetime import datetime

from pydantic import BaseModel, Field


class StudentBase(BaseModel):
    name: str = Field(min_length=1, max_length=150)
    age: int = Field(ge=10, le=100)
    education: str = Field(min_length=1, max_length=150)
    location: str | None = Field(default=None, max_length=150)
    board: str | None = Field(default=None, max_length=50)
    subjects: list[str] | None = None


class StudentCreate(StudentBase):
    pass


class StudentUpdate(BaseModel):
    name: str | None = Field(default=None, min_length=1, max_length=150)
    age: int | None = Field(default=None, ge=10, le=100)
    education: str | None = Field(default=None, min_length=1, max_length=150)
    location: str | None = Field(default=None, max_length=150)
    board: str | None = Field(default=None, max_length=50)
    subjects: list[str] | None = None


class StudentOut(StudentBase):
    student_id: int
    created_date: datetime

    model_config = {"from_attributes": True}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\security\__init__.py
C:\ws\agent\admission\counselor-agent\backend\security\jwt.py
import uuid
from datetime import datetime, timedelta, timezone
from typing import Any

from jose import JWTError, jwt

from config.settings import settings

ACCESS_TOKEN_TYPE = "access"
REFRESH_TOKEN_TYPE = "refresh"
RESET_TOKEN_TYPE = "password_reset"


def _create_token(subject: str, role: str, token_type: str, expires_delta: timedelta) -> str:
    now = datetime.now(timezone.utc)
    payload: dict[str, Any] = {
        "sub": subject,
        "role": role,
        "type": token_type,
        "iat": now,
        "exp": now + expires_delta,
        "jti": str(uuid.uuid4()),
    }
    return jwt.encode(payload, settings.jwt_secret, algorithm=settings.jwt_algorithm)


def create_access_token(user_id: int, role: str) -> str:
    return _create_token(
        str(user_id), role, ACCESS_TOKEN_TYPE, timedelta(minutes=settings.access_token_expire_minutes)
    )


def create_refresh_token(user_id: int, role: str) -> str:
    return _create_token(
        str(user_id), role, REFRESH_TOKEN_TYPE, timedelta(days=settings.refresh_token_expire_days)
    )


def create_password_reset_token(user_id: int, role: str) -> str:
    return _create_token(str(user_id), role, RESET_TOKEN_TYPE, timedelta(minutes=30))


def decode_token(token: str) -> dict:
    try:
        return jwt.decode(token, settings.jwt_secret, algorithms=[settings.jwt_algorithm])
    except JWTError as exc:
        raise ValueError("Invalid or expired token") from exc

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\security\password.py
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")


def hash_password(password: str) -> str:
    return pwd_context.hash(password)


def verify_password(plain_password: str, hashed_password: str) -> bool:
    return pwd_context.verify(plain_password, hashed_password)

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\security\roles.py
ADMIN = "ADMIN"
COUNSELOR = "COUNSELOR"
MENTOR = "MENTOR"
STUDENT = "STUDENT"
READ_ONLY = "READ_ONLY"

ALL_ROLES = [ADMIN, COUNSELOR, MENTOR, STUDENT, READ_ONLY]

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\services\__init__.py
C:\ws\agent\admission\counselor-agent\backend\services\analytics_service.py
from statistics import mean

from datetime import datetime

from sqlalchemy import func
from sqlalchemy.orm import Session

from models.assessment_score import AssessmentScore
from models.career_recommendation import CareerRecommendation
from models.progress_tracking import ProgressTracking
from models.roadmap import Roadmap
from models.roadmap_milestone import RoadmapMilestone
from models.skill import Skill
from models.skill_gap import SkillGap
from models.student import Student
from services.gap_analysis_service import MISSING
from services.market_trend_service import get_career_demand_trends

_CERT_PREFIX = "Certification:"


def career_demand_trends() -> dict:
    return {"chart_type": "bar", "data": get_career_demand_trends()}


def cohort_skill_gap_heatmap(db: Session) -> dict:
    """Counts MISSING skill gaps per (career, skill) across all students."""
    rows = (
        db.query(CareerRecommendation.career_name, Skill.skill_name, func.count(SkillGap.gap_id))
        .join(SkillGap, SkillGap.recommendation_id == CareerRecommendation.recommendation_id)
        .join(Skill, SkillGap.skill_id == Skill.skill_id)
        .filter(SkillGap.status == MISSING)
        .group_by(CareerRecommendation.career_name, Skill.skill_name)
        .all()
    )
    heatmap: dict[str, dict[str, int]] = {}
    for career_name, skill_name, count in rows:
        heatmap.setdefault(career_name, {})[skill_name] = count

    data = [
        {"career_name": career_name, "skill_gap_counts": counts} for career_name, counts in heatmap.items()
    ]
    return {"chart_type": "heatmap", "data": data}


def roadmap_completion_rate(db: Session) -> dict:
    rows = (
        db.query(CareerRecommendation.career_name, ProgressTracking.completion_percentage)
        .join(Roadmap, Roadmap.roadmap_id == ProgressTracking.roadmap_id)
        .join(CareerRecommendation, CareerRecommendation.recommendation_id == Roadmap.recommendation_id)
        .all()
    )
    overall = round(mean([pct for _, pct in rows]), 2) if rows else 0.0

    by_career_values: dict[str, list[float]] = {}
    for career_name, pct in rows:
        by_career_values.setdefault(career_name, []).append(pct)
    by_career = [
        {"career_name": career_name, "average_completion_percentage": round(mean(values), 2)}
        for career_name, values in by_career_values.items()
    ]

    return {
        "chart_type": "bar",
        "overall_average_completion_percentage": overall,
        "by_career": by_career,
    }


def top_recommended_careers(db: Session, limit: int = 5) -> dict:
    rows = (
        db.query(CareerRecommendation.career_name, func.count(CareerRecommendation.recommendation_id))
        .group_by(CareerRecommendation.career_name)
        .order_by(func.count(CareerRecommendation.recommendation_id).desc())
        .limit(limit)
        .all()
    )
    data = [{"career_name": career_name, "recommendation_count": count} for career_name, count in rows]
    return {"chart_type": "pie", "data": data}


def certification_uptake(db: Session) -> dict:
    total_roadmaps = db.query(func.count(Roadmap.roadmap_id)).scalar() or 0
    milestones = db.query(RoadmapMilestone.roadmap_id, RoadmapMilestone.resources, RoadmapMilestone.status).all()

    roadmaps_with_cert = set()
    completed_cert_roadmaps = set()
    for roadmap_id, resources, status in milestones:
        if resources and any(r.startswith(_CERT_PREFIX) for r in resources):
            roadmaps_with_cert.add(roadmap_id)
            if status == "COMPLETED":
                completed_cert_roadmaps.add(roadmap_id)

    return {
        "chart_type": "funnel",
        "data": [
            {"stage": "Roadmaps Generated", "count": total_roadmaps},
            {"stage": "Includes Certification Milestone", "count": len(roadmaps_with_cert)},
            {"stage": "Certification Milestone Completed", "count": len(completed_cert_roadmaps)},
        ],
    }


def dashboard_summary(db: Session) -> dict:
    today_start = datetime.utcnow().replace(hour=0, minute=0, second=0, microsecond=0)
    active_students = db.query(func.count(Student.student_id)).scalar() or 0
    assessments_completed_today = (
        db.query(func.count(AssessmentScore.score_id))
        .filter(AssessmentScore.computed_date >= today_start)
        .scalar()
        or 0
    )
    return {
        "career_demand_trends": career_demand_trends(),
        "cohort_skill_gap_heatmap": cohort_skill_gap_heatmap(db),
        "roadmap_completion_rate": roadmap_completion_rate(db),
        "top_recommended_careers": top_recommended_careers(db),
        "certification_uptake": certification_uptake(db),
        "active_students": active_students,
        "assessments_completed_today": assessments_completed_today,
    }

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\services\assessment_service.py
from statistics import mean

from sqlalchemy.orm import Session

from agents.assessment_agent.career_catalog import CAREER_DEFINITIONS
from models.behavioral_metric import BehavioralMetric
from repositories.academic_history_repository import list_academic_history
from repositories.discovery_repository import list_responses
from repositories.skill_repository import list_behavioral_metrics, list_student_skills
from repositories.student_repository import get_student
from models.discovery_question import DiscoveryQuestion
from services.market_trend_service import get_market_demand_score

_BASELINE_SCORE = 50.0

WEIGHTS = {
    "interest_score": 0.30,
    "skill_score": 0.25,
    "academic_score": 0.15,
    "aptitude_score": 0.15,
    "market_demand_score": 0.15,
}


def _responses_by_category(db: Session, student_id: int) -> dict[str, str]:
    responses = list_responses(db, student_id)
    if not responses:
        return {}
    question_ids = {r.question_id for r in responses}
    questions = db.query(DiscoveryQuestion).filter(DiscoveryQuestion.question_id.in_(question_ids)).all()
    category_by_question = {q.question_id: q.category for q in questions}
    result: dict[str, str] = {}
    for response in responses:
        category = category_by_question.get(response.question_id)
        if category:
            result[category] = response.selected_option
    return result


def _interest_score(answers: dict[str, str], interest_map: dict[str, str]) -> float:
    if not interest_map:
        return _BASELINE_SCORE
    total = 0.0
    for category, preferred_option in interest_map.items():
        answered = answers.get(category)
        if answered == preferred_option:
            total += 100
        elif answered is not None:
            total += 50
        else:
            total += 30
    return round(total / len(interest_map), 2)


def _skill_score(skill_pairs: list, required_skills: list[str]) -> float:
    latest_by_name = {skill.skill_name: student_skill.proficiency_score for student_skill, skill in skill_pairs}
    matched = [latest_by_name[name] for name in required_skills if name in latest_by_name]
    if matched:
        return round(mean(matched), 2)
    return _BASELINE_SCORE


def _academic_score(academic_records, subjects: list[str]) -> float:
    matched = [r.score for r in academic_records if r.subject in subjects]
    if matched:
        return round(mean(matched), 2)
    if academic_records:
        return round(mean([r.score for r in academic_records]), 2)
    return _BASELINE_SCORE


def _aptitude_score(metrics: list[BehavioralMetric]) -> float:
    if not metrics:
        return _BASELINE_SCORE
    latest = metrics[-1]
    values = [
        v
        for v in (
            latest.learning_speed,
            latest.consistency,
            latest.curiosity_index,
            latest.persistence,
            latest.goal_completion_rate,
        )
        if v is not None
    ]
    if not values:
        return _BASELINE_SCORE
    return round(mean(values), 2)


def compute_scores(db: Session, student_id: int, career_names: list[str] | None = None) -> list[dict]:
    """Computes the weighted dimension score for each requested career (default: full catalog)."""
    if not get_student(db, student_id):
        raise ValueError(f"Student {student_id} not found")

    careers = career_names or list(CAREER_DEFINITIONS.keys())
    unknown = [name for name in careers if name not in CAREER_DEFINITIONS]
    if unknown:
        raise KeyError(f"Unknown career(s): {unknown}")

    answers = _responses_by_category(db, student_id)
    skill_pairs = list_student_skills(db, student_id)
    academic_records = list_academic_history(db, student_id)
    metrics = list_behavioral_metrics(db, student_id)

    results = []
    for career_name in careers:
        definition = CAREER_DEFINITIONS[career_name]
        interest = _interest_score(answers, definition["interest_map"])
        skill = _skill_score(skill_pairs, definition["required_skills"])
        academic = _academic_score(academic_records, definition["academic_subjects"])
        aptitude = _aptitude_score(metrics)
        market = get_market_demand_score(career_name)

        final = round(
            interest * WEIGHTS["interest_score"]
            + skill * WEIGHTS["skill_score"]
            + academic * WEIGHTS["academic_score"]
            + aptitude * WEIGHTS["aptitude_score"]
            + market * WEIGHTS["market_demand_score"],
            2,
        )

        results.append(
            {
                "career_name": career_name,
                "interest_score": interest,
                "skill_score": skill,
                "academic_score": academic,
                "aptitude_score": aptitude,
                "market_demand_score": market,
                "final_score": final,
            }
        )

    results.sort(key=lambda r: r["final_score"], reverse=True)
    return results

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\services\auth_service.py
from datetime import datetime, timezone

from sqlalchemy.orm import Session

from models.token_blacklist import TokenBlacklist
from repositories.user_repository import get_user_by_email, get_user_by_id, update_password
from security.jwt import (
    REFRESH_TOKEN_TYPE,
    RESET_TOKEN_TYPE,
    create_access_token,
    create_password_reset_token,
    create_refresh_token,
    decode_token,
)
from security.password import hash_password, verify_password


class AuthError(Exception):
    pass


def authenticate(db: Session, email: str, password: str):
    user = get_user_by_email(db, email)
    if not user or not user.is_active or not verify_password(password, user.hashed_password):
        raise AuthError("Invalid credentials")
    return user


def issue_tokens(user) -> tuple[str, str]:
    role_name = user.role.role_name
    return create_access_token(user.user_id, role_name), create_refresh_token(user.user_id, role_name)


def is_token_blacklisted(db: Session, jti: str) -> bool:
    return db.query(TokenBlacklist).filter(TokenBlacklist.jti == jti).first() is not None


def _blacklist(db: Session, payload: dict) -> None:
    expires_at = datetime.fromtimestamp(payload["exp"], tz=timezone.utc)
    db.merge(TokenBlacklist(jti=payload["jti"], expires_at=expires_at))
    db.commit()


def refresh_access_token(db: Session, refresh_token: str) -> str:
    payload = decode_token(refresh_token)
    if payload.get("type") != REFRESH_TOKEN_TYPE:
        raise AuthError("Invalid refresh token")
    if is_token_blacklisted(db, payload["jti"]):
        raise AuthError("Token has been revoked")
    user = get_user_by_id(db, int(payload["sub"]))
    if not user or not user.is_active:
        raise AuthError("User not found or inactive")
    return create_access_token(user.user_id, user.role.role_name)


def revoke_token(db: Session, token: str) -> None:
    payload = decode_token(token)
    _blacklist(db, payload)


def request_password_reset(db: Session, email: str) -> str | None:
    user = get_user_by_email(db, email)
    if not user:
        return None
    return create_password_reset_token(user.user_id, user.role.role_name)


def confirm_password_reset(db: Session, reset_token: str, new_password: str) -> None:
    payload = decode_token(reset_token)
    if payload.get("type") != RESET_TOKEN_TYPE:
        raise AuthError("Invalid password reset token")
    if is_token_blacklisted(db, payload["jti"]):
        raise AuthError("Token has been revoked")
    user = get_user_by_id(db, int(payload["sub"]))
    if not user:
        raise AuthError("User not found")
    update_password(db, user, hash_password(new_password))
    _blacklist(db, payload)  # one-time use

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\services\gap_analysis_service.py
from sqlalchemy.orm import Session

from agents.assessment_agent.career_catalog import CAREER_DEFINITIONS, infer_skill_type
from repositories.recommendation_repository import get_recommendation
from repositories.skill_gap_repository import (
    create_skill_gap,
    delete_gaps_for_recommendation,
    list_skill_gaps,
)
from repositories.skill_repository import get_or_create_skill, list_student_skills

MATCHED = "MATCHED"
MISSING = "MISSING"


def analyze_gap(db: Session, recommendation_id: int) -> list[dict]:
    """Diffs a student's current skills against their recommended career's target skills."""
    recommendation = get_recommendation(db, recommendation_id)
    if not recommendation:
        raise ValueError(f"Recommendation {recommendation_id} not found")

    career_name = recommendation.career_name
    if career_name not in CAREER_DEFINITIONS:
        raise KeyError(f"Unknown career: {career_name}")

    target_skills = CAREER_DEFINITIONS[career_name]["target_skills"]
    current_skill_names = {skill.skill_name for _, skill in list_student_skills(db, recommendation.student_id)}

    delete_gaps_for_recommendation(db, recommendation_id)

    results = []
    for skill_name in target_skills:
        status = MATCHED if skill_name in current_skill_names else MISSING
        skill = get_or_create_skill(db, skill_name, infer_skill_type(skill_name))
        gap = create_skill_gap(db, recommendation_id, skill.skill_id, status)
        results.append({"gap_id": gap.gap_id, "skill_id": skill.skill_id, "skill_name": skill_name, "status": status})

    return results


def get_gap_analysis(db: Session, recommendation_id: int) -> list[dict]:
    return [
        {"gap_id": gap.gap_id, "skill_id": gap.skill_id, "skill_name": skill_name, "status": gap.status}
        for gap, skill_name in list_skill_gaps(db, recommendation_id)
    ]

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\services\gemini_service.py
import logging
import time

import google.generativeai as genai

from config.settings import settings

logger = logging.getLogger("gemini_service")

_MAX_RETRIES = 3
_RETRY_BACKOFF_SECONDS = 1.0
_EMBEDDING_MODEL = "models/text-embedding-004"

_configured = False


class GeminiServiceError(Exception):
    pass


def _ensure_configured() -> None:
    global _configured
    if not _configured:
        if not settings.gemini_api_key:
            raise GeminiServiceError("GEMINI_API_KEY is not configured")
        genai.configure(api_key=settings.gemini_api_key)
        _configured = True


def _truncate_to_word_limit(prompt: str, max_words: int) -> str:
    words = prompt.split()
    if len(words) <= max_words:
        return prompt
    logger.warning("gemini_prompt_truncated", extra={"original_words": len(words), "max_words": max_words})
    return " ".join(words[:max_words])


def generate_text(prompt: str, model: str | None = None) -> str:
    """Sends `prompt` to Gemini and returns the generated text, retrying on transient errors."""
    _ensure_configured()
    model_name = model or settings.gemini_model
    prompt = _truncate_to_word_limit(prompt, settings.gemini_max_input_words)
    generation_config = {
        "temperature": settings.gemini_temperature,
        "max_output_tokens": settings.gemini_max_output_tokens,
    }
    logger.info("gemini_request", extra={"model": model_name, "prompt_length": len(prompt)})

    last_error: Exception | None = None
    for attempt in range(1, _MAX_RETRIES + 1):
        try:
            client = genai.GenerativeModel(model_name, generation_config=generation_config)
            response = client.generate_content(prompt)
            text = response.text
            logger.info("gemini_response", extra={"model": model_name, "response_length": len(text)})
            return text
        except Exception as exc:  # noqa: BLE001 - any SDK/network error is retried, then wrapped below
            last_error = exc
            logger.warning("gemini_retry", extra={"attempt": attempt, "error": str(exc)})
            if attempt < _MAX_RETRIES:
                time.sleep(_RETRY_BACKOFF_SECONDS * attempt)

    logger.error("gemini_failed", extra={"model": model_name, "error": str(last_error)})
    raise GeminiServiceError(f"Gemini request failed after {_MAX_RETRIES} attempts: {last_error}") from last_error


def embed_text(text: str, model: str = _EMBEDDING_MODEL) -> list[float]:
    """Embeds `text` into a vector using Gemini's embedding model, retrying on transient errors."""
    _ensure_configured()

    last_error: Exception | None = None
    for attempt in range(1, _MAX_RETRIES + 1):
        try:
            result = genai.embed_content(model=model, content=text)
            return result["embedding"]
        except Exception as exc:  # noqa: BLE001
            last_error = exc
            logger.warning("gemini_embed_retry", extra={"attempt": attempt, "error": str(exc)})
            if attempt < _MAX_RETRIES:
                time.sleep(_RETRY_BACKOFF_SECONDS * attempt)

    raise GeminiServiceError(f"Gemini embedding failed after {_MAX_RETRIES} attempts: {last_error}") from last_error

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\services\market_trend_service.py
from agents.assessment_agent.career_catalog import CAREER_DEFINITIONS

# Stub for the "Job Market MCP" integration: currently reads the static catalog
# placeholder: swap this out for a live market-data API call without touching callers.


def get_market_demand_score(career_name: str) -> float:
    return CAREER_DEFINITIONS[career_name]["market_demand_score"]


def get_career_demand_trends() -> list[dict]:
    trends = [
        {"career_name": name, "market_demand_score": get_market_demand_score(name)}
        for name in CAREER_DEFINITIONS
    ]
    return sorted(trends, key=lambda t: t["market_demand_score"], reverse=True)

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\services\mentor_service.py
from sqlalchemy.orm import Session

from repositories.mentor_repository import create_conversation
from repositories.recommendation_repository import list_recommendations
from repositories.skill_gap_repository import list_skill_gaps
from repositories.student_repository import get_student
from services.gap_analysis_service import MATCHED
from services.gemini_service import GeminiServiceError, generate_text


def _top_recommendation(db: Session, student_id: int):
    recommendations = list_recommendations(db, student_id)
    if not recommendations:
        return None
    return max(recommendations, key=lambda r: r.match_score)


def _fallback_reply(top_recommendation, matched: list[str], missing: list[str]) -> str:
    if not top_recommendation:
        return "I don't have any career recommendations for you yet - let's run an assessment first."
    matched_str = ", ".join(matched) if matched else "no confirmed skills yet"
    missing_str = ", ".join(missing) if missing else "no major gaps"
    return (
        f"Based on your assessment, {top_recommendation.career_name} is currently your strongest match "
        f"({top_recommendation.match_score}/100). You already have {matched_str} covered; "
        f"focus next on {missing_str} to strengthen this path."
    )


def chat(db: Session, student_id: int, message: str) -> dict:
    """Answers a student's question, grounded in their actual recommendations and skill gaps."""
    if not get_student(db, student_id):
        raise ValueError(f"Student {student_id} not found")

    top_recommendation = _top_recommendation(db, student_id)
    matched: list[str] = []
    missing: list[str] = []
    if top_recommendation:
        for gap, skill_name in list_skill_gaps(db, top_recommendation.recommendation_id):
            (matched if gap.status == MATCHED else missing).append(skill_name)

    if top_recommendation:
        prompt = (
            f"You are a career mentor. The student asked: {message!r}. "
            f"Their top career match is {top_recommendation.career_name} "
            f"(match score {top_recommendation.match_score}/100). "
            f"Skills they already have: {matched or 'none recorded'}. "
            f"Skills they are missing: {missing or 'none recorded'}. "
            "Reply conversationally in 2-4 sentences, referencing this data directly."
        )
        try:
            reply = generate_text(prompt)
        except GeminiServiceError:
            reply = _fallback_reply(top_recommendation, matched, missing)
    else:
        reply = _fallback_reply(top_recommendation, matched, missing)

    related_career = top_recommendation.career_name if top_recommendation else None
    create_conversation(db, student_id, message, reply, related_career)
    return {"reply": reply, "related_career": related_career}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\services\progress_service.py
from datetime import datetime

from sqlalchemy.orm import Session

from repositories.assessment_repository import save_assessment_score
from repositories.progress_repository import (
    create_progress,
    get_progress_by_roadmap,
    list_progress_by_student,
    update_progress,
)
from repositories.recommendation_repository import create_recommendation, get_recommendation
from repositories.roadmap_repository import get_milestone, get_roadmap, list_milestones
from services.assessment_service import compute_scores
from services.recommendation_service import generate_recommendations

COMPLETED = "COMPLETED"


def _milestone_counts(db: Session, roadmap_id: int) -> tuple[int, int]:
    milestones = list_milestones(db, roadmap_id)
    completed = sum(1 for m in milestones if m.status == COMPLETED)
    return completed, len(milestones)


def complete_milestone(db: Session, milestone_id: int) -> dict:
    """Marks a milestone complete, recomputes roadmap progress, and re-runs Phase 8/9 scoring."""
    milestone = get_milestone(db, milestone_id)
    if not milestone:
        raise ValueError(f"Milestone {milestone_id} not found")

    roadmap = get_roadmap(db, milestone.roadmap_id)
    recommendation = get_recommendation(db, roadmap.recommendation_id)

    already_completed = milestone.status == COMPLETED
    if not already_completed:
        milestone.status = COMPLETED
        milestone.completed_date = datetime.utcnow()
        db.commit()

    completed_count, total = _milestone_counts(db, roadmap.roadmap_id)
    percentage = round(completed_count / total * 100, 2) if total else 0.0

    progress = get_progress_by_roadmap(db, roadmap.roadmap_id)
    if not progress:
        progress = create_progress(db, roadmap.student_id, roadmap.roadmap_id)

    updated_recommendation = None
    reassessment_triggered = False
    last_reassessed_date = progress.last_reassessed_date

    if not already_completed:
        reassessment_triggered = True
        scores = compute_scores(db, roadmap.student_id, [recommendation.career_name])
        for r in scores:
            dimension_scores = {k: v for k, v in r.items() if k != "career_name"}
            save_assessment_score(db, roadmap.student_id, r["career_name"], dimension_scores)

        recs = generate_recommendations(db, roadmap.student_id, [recommendation.career_name])
        for r in recs:
            updated_recommendation = create_recommendation(
                db, roadmap.student_id, r["career_name"], r["match_score"], r["rationale"]
            )
        last_reassessed_date = datetime.utcnow()

    progress = update_progress(db, progress, completed_count, percentage, last_reassessed_date)

    return {
        "progress": progress,
        "total_milestones": total,
        "reassessment_triggered": reassessment_triggered,
        "updated_recommendation": updated_recommendation,
    }


def get_progress_for_student(db: Session, student_id: int) -> list[dict]:
    entries = []
    for progress in list_progress_by_student(db, student_id):
        _, total = _milestone_counts(db, progress.roadmap_id)
        entries.append({"progress": progress, "total_milestones": total})
    return entries

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\services\recommendation_service.py
from sqlalchemy.orm import Session

from agents.assessment_agent.career_catalog import CAREER_DEFINITIONS
from services.assessment_service import WEIGHTS, compute_scores
from services.gemini_service import GeminiServiceError, generate_text

_DIMENSION_LABELS = {
    "interest_score": "interest fit",
    "skill_score": "current skills",
    "academic_score": "academic history",
    "aptitude_score": "aptitude/behavioral traits",
    "market_demand_score": "market demand",
}


def _top_dimension_label(score: dict) -> str:
    weighted = {dim: score[dim] * WEIGHTS[dim] for dim in WEIGHTS}
    top_dimension = max(weighted, key=weighted.get)
    return _DIMENSION_LABELS[top_dimension]


def _fallback_rationale(score: dict) -> str:
    label = _top_dimension_label(score)
    return (
        f"{score['career_name']} is a {score['final_score']}/100 match, driven primarily by {label} "
        f"(interest {score['interest_score']}, skill {score['skill_score']}, academic {score['academic_score']}, "
        f"aptitude {score['aptitude_score']}, market demand {score['market_demand_score']})."
    )


def _generate_rationale(score: dict) -> str:
    prompt = (
        f"In 2-3 sentences, explain why {score['career_name']} suits a student with these dimension scores "
        f"(0-100): interest {score['interest_score']}, skill {score['skill_score']}, "
        f"academic {score['academic_score']}, aptitude {score['aptitude_score']}, "
        f"market demand {score['market_demand_score']}, overall match {score['final_score']}."
    )
    try:
        return generate_text(prompt)
    except GeminiServiceError:
        return _fallback_rationale(score)


def generate_recommendations(
    db: Session, student_id: int, career_names: list[str] | None = None, top_n: int | None = None
) -> list[dict]:
    """Rule-based dimension scoring (Phase 8) ranked and explained via an LLM-generated rationale."""
    scores = compute_scores(db, student_id, career_names)
    if top_n:
        scores = scores[:top_n]

    return [
        {
            "career_name": s["career_name"],
            "match_score": s["final_score"],
            "rationale": _generate_rationale(s),
        }
        for s in scores
    ]


ALL_CAREER_NAMES = list(CAREER_DEFINITIONS.keys())

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\services\roadmap_service.py
import math
from itertools import cycle

from sqlalchemy.orm import Session

from agents.assessment_agent.career_catalog import CAREER_DEFINITIONS
from repositories.recommendation_repository import get_recommendation
from repositories.roadmap_repository import (
    create_milestone,
    create_roadmap,
    get_roadmap,
    list_certifications_by_career,
    list_milestones,
    list_projects_by_career,
)
from services.gap_analysis_service import MISSING, analyze_gap

_MAX_PHASES = 6
_RESOURCE_POOL = ["Coursera", "Udemy", "YouTube Learning", "LinkedIn Learning", "Coding Platforms"]


def _chunk(items: list, n: int) -> list[list]:
    k, m = divmod(len(items), n)
    return [items[i * k + min(i, m) : (i + 1) * k + min(i + 1, m)] for i in range(n)]


def _month_ranges(duration_months: int, phase_count: int) -> list[str]:
    months_per_phase, remainder = divmod(duration_months, phase_count)
    ranges = []
    start = 1
    for i in range(phase_count):
        length = months_per_phase + (1 if i >= phase_count - remainder else 0)
        end = start + length - 1
        ranges.append(f"{start}-{end}")
        start = end + 1
    return ranges


def generate_roadmap(db: Session, recommendation_id: int, duration_months: int = 12) -> dict:
    """Builds a multi-phase learning roadmap from the student's missing skills for a recommended career."""
    recommendation = get_recommendation(db, recommendation_id)
    if not recommendation:
        raise ValueError(f"Recommendation {recommendation_id} not found")

    career_name = recommendation.career_name
    if career_name not in CAREER_DEFINITIONS:
        raise KeyError(f"Unknown career: {career_name}")

    gap_results = analyze_gap(db, recommendation_id)
    topics_pool = [r["skill_name"] for r in gap_results if r["status"] == MISSING]
    if not topics_pool:
        topics_pool = CAREER_DEFINITIONS[career_name]["target_skills"]

    phase_count = max(1, min(_MAX_PHASES, math.ceil(duration_months / 2), len(topics_pool)))
    topic_chunks = _chunk(topics_pool, phase_count)
    month_ranges = _month_ranges(duration_months, phase_count)

    certifications = list_certifications_by_career(db, career_name)
    projects = list_projects_by_career(db, career_name)
    resource_cycle = cycle(_RESOURCE_POOL)

    roadmap = create_roadmap(db, recommendation.student_id, recommendation_id, f"Become {career_name}", duration_months)

    milestones = []
    for i in range(phase_count):
        resources = [next(resource_cycle), next(resource_cycle)]
        if i < len(certifications):
            resources.append(f"Certification: {certifications[i].certification_name}")
        project = projects[i % len(projects)].project_name if projects else None

        milestone = create_milestone(db, roadmap.roadmap_id, month_ranges[i], topic_chunks[i], resources, project)
        milestones.append(milestone)

    return {"roadmap": roadmap, "milestones": milestones}


def get_roadmap_with_milestones(db: Session, roadmap_id: int) -> dict | None:
    roadmap = get_roadmap(db, roadmap_id)
    if not roadmap:
        return None
    return {"roadmap": roadmap, "milestones": list_milestones(db, roadmap_id)}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\services\vector_store.py
from sqlalchemy.orm import Session

from models.embedding import Embedding
from services.gemini_service import embed_text

CAREER = "CAREER"
CERTIFICATION = "CERTIFICATION"
COURSE = "COURSE"
PROJECT = "PROJECT"
JOB_MARKET_ROLE = "JOB_MARKET_ROLE"
FAQ = "FAQ"

ENTITY_TYPES = [CAREER, CERTIFICATION, COURSE, PROJECT, JOB_MARKET_ROLE, FAQ]


def store_embedding(db: Session, entity_type: str, content: str, entity_id: int | None = None) -> Embedding:
    """Embeds `content` via Gemini and persists it for later semantic/similarity search."""
    vector = embed_text(content)
    embedding = Embedding(entity_type=entity_type, entity_id=entity_id, content=content, vector=vector)
    db.add(embedding)
    db.commit()
    db.refresh(embedding)
    return embedding


def similarity_search(
    db: Session, vector: list[float], entity_type: str | None = None, top_k: int = 5
) -> list[Embedding]:
    """Finds the closest stored embeddings to a given vector using cosine distance (requires pgvector)."""
    query = db.query(Embedding)
    if entity_type:
        query = query.filter(Embedding.entity_type == entity_type)
    return query.order_by(Embedding.vector.cosine_distance(vector)).limit(top_k).all()


def semantic_search(
    db: Session, query_text: str, entity_type: str | None = None, top_k: int = 5
) -> list[Embedding]:
    """Embeds `query_text` via Gemini and finds the closest stored embeddings."""
    query_vector = embed_text(query_text)
    return similarity_search(db, query_vector, entity_type, top_k)

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\utils\__init__.py

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\workflows\__init__.py

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\alembic.ini
[alembic]
script_location = database/migrations
prepend_sys_path = .
sqlalchemy.url =

[loggers]
keys = root,sqlalchemy,alembic

[handlers]
keys = console

[formatters]
keys = generic

[logger_root]
level = WARNING
handlers = console
qualname =

[logger_sqlalchemy]
level = WARNING
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

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\Dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY backend/ .

EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\backend\main.py
from fastapi import FastAPI

from api.routes.academic_learning_history import router as academic_learning_history_router
from api.routes.ai_diagnostics import router as ai_diagnostics_router
from api.routes.assessment import router as assessment_router
from api.routes.auth import router as auth_router
from api.routes.catalog import router as catalog_router
from api.routes.discovery import router as discovery_router
from api.routes.analytics import router as analytics_router
from api.routes.gap_analysis import router as gap_analysis_router
from api.routes.health import router as health_router
from api.routes.mentor import router as mentor_router
from api.routes.orchestrator import router as orchestrator_router
from api.routes.progress import router as progress_router
from api.routes.prompts import router as prompts_router
from api.routes.recommendations import router as recommendations_router
from api.routes.roadmaps import router as roadmaps_router
from api.routes.skills import router as skills_router
from api.routes.students import router as students_router
from api.routes.users import router as users_router

app = FastAPI(title="Career Counselor Agent", version="0.1.0")

app.include_router(health_router)
app.include_router(auth_router)
app.include_router(users_router)
app.include_router(students_router)
app.include_router(academic_learning_history_router)
app.include_router(skills_router)
app.include_router(discovery_router)
app.include_router(assessment_router)
app.include_router(recommendations_router)
app.include_router(gap_analysis_router)
app.include_router(roadmaps_router)
app.include_router(mentor_router)
app.include_router(progress_router)
app.include_router(analytics_router)
app.include_router(catalog_router)
app.include_router(orchestrator_router)
app.include_router(prompts_router)
app.include_router(ai_diagnostics_router)
=========================================================
C:\ws\agent\admission\counselor-agent\frontend\public\.gitkeep
-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\core\auth\auth.guard.ts
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';

import { AuthService } from './auth.service';

export const authGuard: CanActivateFn = (_route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);

  if (authService.isAuthenticated()) {
    return true;
  }
  return router.createUrlTree(['/login'], { queryParams: { returnUrl: state.url } });
};

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\core\auth\auth.interceptor.ts
import { HttpErrorResponse, HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';
import { catchError, switchMap, throwError } from 'rxjs';

import { AuthService } from './auth.service';

export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const authService = inject(AuthService);
  const token = authService.getAccessToken();
  const authorizedReq = token ? req.clone({ setHeaders: { Authorization: `Bearer ${token}` } }) : req;

  return next(authorizedReq).pipe(
    catchError((error: HttpErrorResponse) => {
      const isAuthEndpoint = req.url.includes('/auth/login') || req.url.includes('/auth/refresh');
      if (error.status === 401 && !isAuthEndpoint && authService.getRefreshToken()) {
        return authService.refreshAccessToken().pipe(
          switchMap((refreshed) => {
            const retriedReq = req.clone({ setHeaders: { Authorization: `Bearer ${refreshed.access_token}` } });
            return next(retriedReq);
          }),
          catchError((refreshError) => {
            authService.logout();
            return throwError(() => refreshError);
          }),
        );
      }
      return throwError(() => error);
    }),
  );
};

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\core\auth\auth.service.ts
import { HttpClient } from '@angular/common/http';
import { Injectable, computed, inject, signal } from '@angular/core';
import { Router } from '@angular/router';
import { Observable, catchError, map, of, switchMap, tap } from 'rxjs';
import { firstValueFrom } from 'rxjs';

import { environment } from '../../../environments/environment';
import { AccessTokenResponse, CurrentUser, LoginRequest, TokenResponse } from '../../models/auth.model';

const ACCESS_TOKEN_KEY = 'counselor.access_token';
const REFRESH_TOKEN_KEY = 'counselor.refresh_token';

@Injectable({ providedIn: 'root' })
export class AuthService {
  private readonly http = inject(HttpClient);
  private readonly router = inject(Router);

  private readonly currentUserSignal = signal<CurrentUser | null>(null);
  readonly currentUser = this.currentUserSignal.asReadonly();
  readonly isAuthenticated = computed(() => this.currentUserSignal() !== null);

  getAccessToken(): string | null {
    return localStorage.getItem(ACCESS_TOKEN_KEY);
  }

  getRefreshToken(): string | null {
    return localStorage.getItem(REFRESH_TOKEN_KEY);
  }

  login(credentials: LoginRequest): Observable<CurrentUser> {
    return this.http.post<TokenResponse>(`${environment.apiUrl}/auth/login`, credentials).pipe(
      tap((tokens) => this.storeTokens(tokens)),
      switchMap(() => this.loadCurrentUser()),
    );
  }

  loadCurrentUser(): Observable<CurrentUser> {
    return this.http
      .get<CurrentUser>(`${environment.apiUrl}/auth/me`)
      .pipe(tap((user) => this.currentUserSignal.set(user)));
  }

  refreshAccessToken(): Observable<AccessTokenResponse> {
    const refreshToken = this.getRefreshToken();
    return this.http
      .post<AccessTokenResponse>(`${environment.apiUrl}/auth/refresh`, { refresh_token: refreshToken })
      .pipe(tap((tokens) => localStorage.setItem(ACCESS_TOKEN_KEY, tokens.access_token)));
  }

  logout(): void {
    this.clearTokens();
    this.currentUserSignal.set(null);
    this.router.navigate(['/login']);
  }

  hasAnyRole(...roles: string[]): boolean {
    const user = this.currentUserSignal();
    return !!user && roles.includes(user.role);
  }

  /** Restores the session from a stored token on app bootstrap; never rejects. */
  restoreSession(): Promise<void> {
    const token = this.getAccessToken();
    if (!token) {
      return Promise.resolve();
    }
    return firstValueFrom(
      this.loadCurrentUser().pipe(
        map(() => void 0),
        catchError(() => {
          this.clearTokens();
          return of(void 0);
        }),
      ),
    );
  }

  private storeTokens(tokens: TokenResponse): void {
    localStorage.setItem(ACCESS_TOKEN_KEY, tokens.access_token);
    localStorage.setItem(REFRESH_TOKEN_KEY, tokens.refresh_token);
  }

  private clearTokens(): void {
    localStorage.removeItem(ACCESS_TOKEN_KEY);
    localStorage.removeItem(REFRESH_TOKEN_KEY);
  }
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\core\auth\role.guard.ts
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';

import { AuthService } from './auth.service';

export function roleGuard(...allowedRoles: string[]): CanActivateFn {
  return () => {
    const authService = inject(AuthService);
    const router = inject(Router);

    if (authService.hasAnyRole(...allowedRoles)) {
      return true;
    }
    return router.createUrlTree(['/unauthorized']);
  };
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\core\.gitkeep
C:\ws\agent\admission\counselor-agent\frontend\src\app\core\roles.ts
export const ROLES = {
  ADMIN: 'ADMIN',
  COUNSELOR: 'COUNSELOR',
  MENTOR: 'MENTOR',
  STUDENT: 'STUDENT',
  READ_ONLY: 'READ_ONLY',
} as const;

export type Role = (typeof ROLES)[keyof typeof ROLES];

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\auth\login\login.css
.login-page {
  display: flex;
  align-items: center;
  justify-content: center;
  min-height: 100vh;
  background: #f3f4f6;
}

.login-card {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  width: 320px;
  padding: 2rem;
  background: #fff;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.login-card h1 {
  font-size: 1.1rem;
  margin: 0;
}

.login-card h2 {
  margin: 0 0 1rem;
  font-size: 1.4rem;
}

.login-card label {
  font-size: 0.85rem;
  font-weight: 600;
}

.login-card input {
  padding: 0.5rem;
  border: 1px solid #d1d5db;
  border-radius: 4px;
  margin-bottom: 0.5rem;
}

.login-card button {
  margin-top: 0.5rem;
  padding: 0.6rem;
  background: #2563eb;
  color: #fff;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.login-card button:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.error {
  color: #dc2626;
  font-size: 0.85rem;
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\auth\login\login.html
<div class="login-page">
  <form class="login-card" [formGroup]="form" (ngSubmit)="onSubmit()">
    <h1>Career Counselor Agent</h1>
    <h2>Sign in</h2>

    @if (errorMessage()) {
      <p class="error">{{ errorMessage() }}</p>
    }

    <label for="email">Email</label>
    <input id="email" type="email" formControlName="email" autocomplete="username" />

    <label for="password">Password</label>
    <input id="password" type="password" formControlName="password" autocomplete="current-password" />

    <button type="submit" [disabled]="submitting()">
      {{ submitting() ? 'Signing in...' : 'Sign in' }}
    </button>
  </form>
</div>

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\auth\login\login.ts
import { Component, inject, signal } from '@angular/core';
import { FormBuilder, ReactiveFormsModule, Validators } from '@angular/forms';
import { ActivatedRoute, Router } from '@angular/router';

import { AuthService } from '../../../core/auth/auth.service';
import { ROLES } from '../../../core/roles';

@Component({
  selector: 'app-login',
  standalone: true,
  imports: [ReactiveFormsModule],
  templateUrl: './login.html',
  styleUrl: './login.css',
})
export class Login {
  private readonly fb = inject(FormBuilder);
  private readonly authService = inject(AuthService);
  private readonly router = inject(Router);
  private readonly route = inject(ActivatedRoute);

  protected readonly form = this.fb.group({
    email: ['', [Validators.required, Validators.email]],
    password: ['', [Validators.required]],
  });

  protected readonly submitting = signal(false);
  protected readonly errorMessage = signal<string | null>(null);

  protected onSubmit(): void {
    if (this.form.invalid) {
      this.form.markAllAsTouched();
      return;
    }

    this.submitting.set(true);
    this.errorMessage.set(null);

    const { email, password } = this.form.getRawValue();
    this.authService.login({ email: email ?? '', password: password ?? '' }).subscribe({
      next: (user) => {
        this.submitting.set(false);
        const returnUrl = this.route.snapshot.queryParamMap.get('returnUrl');
        const destination = returnUrl ?? (user.role === ROLES.STUDENT ? '/student' : '/counselor');
        this.router.navigateByUrl(destination);
      },
      error: () => {
        this.submitting.set(false);
        this.errorMessage.set('Invalid email or password.');
      },
    });
  }
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\counselor-portal\analytics\analytics.css
.bar-list {
  list-style: none;
  padding: 0;
  max-width: 600px;
  margin-bottom: 1.5rem;
}

.bar-list li {
  display: grid;
  grid-template-columns: 160px 1fr 48px;
  align-items: center;
  gap: 0.5rem;
  margin-bottom: 0.5rem;
}

.bar-label {
  font-size: 0.85rem;
  color: #374151;
}

.bar {
  height: 10px;
  background: #e5e7eb;
  border-radius: 5px;
  overflow: hidden;
}

.bar-fill {
  height: 100%;
  background: #2563eb;
}

.bar-value {
  font-size: 0.85rem;
  text-align: right;
}

.overall-rate {
  margin-bottom: 0.75rem;
}

.heatmap-table {
  border-collapse: collapse;
  margin-bottom: 1.5rem;
  max-width: 100%;
  overflow-x: auto;
  display: block;
}

.heatmap-table th,
.heatmap-table td {
  padding: 0.4rem 0.6rem;
  border: 1px solid #e5e7eb;
  text-align: center;
  font-size: 0.85rem;
}

.heatmap-table th:first-child,
.heatmap-table td:first-child {
  text-align: left;
  font-weight: 600;
}

.error {
  color: #dc2626;
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\counselor-portal\analytics\analytics.html
<h2>Analytics & Reports</h2>

@if (loading()) {
  <p>Loading...</p>
} @else if (error()) {
  <p class="error">{{ error() }}</p>
}

<h3>Career Demand Trends</h3>
@if (demandTrends()?.data?.length) {
  <ul class="bar-list">
    @for (d of demandTrends()!.data; track d.career_name) {
      <li>
        <span class="bar-label">{{ d.career_name }}</span>
        <div class="bar">
          <div class="bar-fill" [style.width.%]="(d.market_demand_score / maxDemandScore()) * 100"></div>
        </div>
        <span class="bar-value">{{ d.market_demand_score }}</span>
      </li>
    }
  </ul>
} @else {
  <p>No career demand data available.</p>
}

<h3>Top Recommended Careers</h3>
@if (topCareers()?.data?.length) {
  <ul class="bar-list">
    @for (c of topCareers()!.data; track c.career_name) {
      <li>
        <span class="bar-label">{{ c.career_name }}</span>
        <div class="bar">
          <div class="bar-fill" [style.width.%]="(c.recommendation_count / maxRecommendationCount()) * 100"></div>
        </div>
        <span class="bar-value">{{ c.recommendation_count }}</span>
      </li>
    }
  </ul>
} @else {
  <p>No recommendation data available.</p>
}

<h3>Roadmap Completion Rate</h3>
@if (completionRate(); as rate) {
  <p class="overall-rate">
    Overall average completion: <strong>{{ rate.overall_average_completion_percentage }}%</strong>
  </p>
  @if (rate.by_career.length) {
    <ul class="bar-list">
      @for (c of rate.by_career; track c.career_name) {
        <li>
          <span class="bar-label">{{ c.career_name }}</span>
          <div class="bar">
            <div class="bar-fill" [style.width.%]="c.average_completion_percentage"></div>
          </div>
          <span class="bar-value">{{ c.average_completion_percentage }}%</span>
        </li>
      }
    </ul>
  }
} @else {
  <p>No roadmap completion data available.</p>
}

<h3>Cohort Skill Gap Heatmap</h3>
@if (heatmap()?.data?.length) {
  <table class="heatmap-table">
    <thead>
      <tr>
        <th>Career</th>
        @for (skill of heatmapSkills(); track skill) {
          <th>{{ skill }}</th>
        }
      </tr>
    </thead>
    <tbody>
      @for (entry of heatmap()!.data; track entry.career_name) {
        <tr>
          <td>{{ entry.career_name }}</td>
          @for (skill of heatmapSkills(); track skill) {
            <td>{{ entry.skill_gap_counts[skill] ?? 0 }}</td>
          }
        </tr>
      }
    </tbody>
  </table>
} @else {
  <p>No skill gap data available.</p>
}

<h3>Certification Uptake Funnel</h3>
@if (uptake()?.data?.length) {
  <ul class="bar-list">
    @for (stage of uptake()!.data; track stage.stage) {
      <li>
        <span class="bar-label">{{ stage.stage }}</span>
        <div class="bar">
          <div class="bar-fill" [style.width.%]="(stage.count / maxFunnelCount()) * 100"></div>
        </div>
        <span class="bar-value">{{ stage.count }}</span>
      </li>
    }
  </ul>
} @else {
  <p>No certification uptake data available.</p>
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\counselor-portal\analytics\analytics.ts
import { Component, inject, signal } from '@angular/core';

import {
  CareerDemandTrends,
  CertificationUptake,
  RoadmapCompletionRate,
  SkillGapHeatmap,
  TopRecommendedCareers,
} from '../../../models/analytics.model';
import { AnalyticsService } from '../../../services/analytics.service';

@Component({
  selector: 'app-analytics',
  standalone: true,
  templateUrl: './analytics.html',
  styleUrl: './analytics.css',
})
export class Analytics {
  private readonly analyticsService = inject(AnalyticsService);

  protected readonly demandTrends = signal<CareerDemandTrends | null>(null);
  protected readonly heatmap = signal<SkillGapHeatmap | null>(null);
  protected readonly completionRate = signal<RoadmapCompletionRate | null>(null);
  protected readonly topCareers = signal<TopRecommendedCareers | null>(null);
  protected readonly uptake = signal<CertificationUptake | null>(null);
  protected readonly loading = signal(true);
  protected readonly error = signal<string | null>(null);

  constructor() {
    this.analyticsService.getCareerDemandTrends().subscribe({
      next: (v) => this.demandTrends.set(v),
      error: () => this.error.set('Failed to load career demand trends.'),
    });

    this.analyticsService.getSkillGapHeatmap().subscribe({
      next: (v) => this.heatmap.set(v),
      error: () => this.error.set('Failed to load the skill gap heatmap.'),
    });

    this.analyticsService.getRoadmapCompletionRate().subscribe({
      next: (v) => this.completionRate.set(v),
      error: () => this.error.set('Failed to load roadmap completion rates.'),
    });

    this.analyticsService.getTopRecommendedCareers().subscribe({
      next: (v) => this.topCareers.set(v),
      error: () => this.error.set('Failed to load top recommended careers.'),
    });

    this.analyticsService.getCertificationUptake().subscribe({
      next: (v) => {
        this.uptake.set(v);
        this.loading.set(false);
      },
      error: () => {
        this.error.set('Failed to load certification uptake.');
        this.loading.set(false);
      },
    });
  }

  protected maxDemandScore(): number {
    const data = this.demandTrends()?.data ?? [];
    return Math.max(1, ...data.map((d) => d.market_demand_score));
  }

  protected maxRecommendationCount(): number {
    const data = this.topCareers()?.data ?? [];
    return Math.max(1, ...data.map((d) => d.recommendation_count));
  }

  protected maxFunnelCount(): number {
    const data = this.uptake()?.data ?? [];
    return Math.max(1, ...data.map((d) => d.count));
  }

  protected heatmapSkills(): string[] {
    const entries = this.heatmap()?.data ?? [];
    const skills = new Set<string>();
    entries.forEach((entry) => Object.keys(entry.skill_gap_counts).forEach((s) => skills.add(s)));
    return Array.from(skills);
  }
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\counselor-portal\career-criteria\career-criteria.css
.read-only-badge {
  font-size: 0.7rem;
  background: #f3f4f6;
  color: #6b7280;
  border-radius: 4px;
  padding: 0.15rem 0.5rem;
  vertical-align: middle;
  margin-left: 0.5rem;
}

.hint {
  color: #6b7280;
  font-size: 0.85rem;
  max-width: 640px;
}

.career-list {
  list-style: none;
  padding: 0;
  max-width: 700px;
}

.career-list > li {
  padding: 0.75rem 1rem;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  margin-bottom: 0.75rem;
}

.career-header {
  display: flex;
  justify-content: space-between;
  margin-bottom: 0.5rem;
}

.career-name {
  font-weight: 600;
}

.demand-score {
  color: #6b7280;
  font-size: 0.85rem;
}

.catalog-form {
  display: flex;
  flex-wrap: wrap;
  gap: 1rem;
  padding: 1rem;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  margin: 1rem 0;
  background: #f9fafb;
  max-width: 700px;
}

.catalog-form label {
  display: flex;
  flex-direction: column;
  font-size: 0.85rem;
  color: #4b5563;
  gap: 0.25rem;
}

.catalog-form input {
  padding: 0.4rem;
  border: 1px solid #d1d5db;
  border-radius: 4px;
}

.catalog-table {
  width: 100%;
  border-collapse: collapse;
  max-width: 700px;
  margin-bottom: 1.5rem;
}

.catalog-table th,
.catalog-table td {
  text-align: left;
  padding: 0.5rem 0.75rem;
  border-bottom: 1px solid #e5e7eb;
}

button.danger {
  color: #dc2626;
  background: none;
  border: 1px solid #dc2626;
  border-radius: 4px;
  padding: 0.2rem 0.5rem;
  cursor: pointer;
}

.error {
  color: #dc2626;
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\counselor-portal\career-criteria\career-criteria.html
<h2>Career Criteria Management</h2>

@if (loading()) {
  <p>Loading...</p>
} @else if (error()) {
  <p class="error">{{ error() }}</p>
}

<h3>Career Definitions <span class="read-only-badge">Read-only</span></h3>
<p class="hint">
  Career criteria (required skills, target skills, interest mapping, market demand) are defined in the
  scoring engine and shown here for reference only.
</p>
@if (careers().length) {
  <ul class="career-list">
    @for (c of careers(); track c.career_name) {
      <li>
        <div class="career-header">
          <span class="career-name">{{ c.career_name }}</span>
          <span class="demand-score">Market Demand: {{ c.market_demand_score }}</span>
        </div>
        <p><strong>Required Skills:</strong> {{ c.required_skills.join(', ') }}</p>
        <p><strong>Target Skills:</strong> {{ c.target_skills.join(', ') }}</p>
        <p><strong>Academic Subjects:</strong> {{ c.academic_subjects.join(', ') }}</p>
      </li>
    }
  </ul>
} @else if (!loading()) {
  <p>No career definitions available.</p>
}

<h3>Certifications Catalog</h3>
@if (canWrite) {
  <button type="button" (click)="toggleCertForm()">
    {{ showCertForm() ? 'Cancel' : 'Add Certification' }}
  </button>

  @if (showCertForm()) {
    <form class="catalog-form" [formGroup]="certForm" (ngSubmit)="createCertification()">
      <label>
        Career Name
        <input type="text" formControlName="career_name" />
      </label>
      <label>
        Certification Name
        <input type="text" formControlName="certification_name" />
      </label>
      <label>
        Provider (optional)
        <input type="text" formControlName="provider" />
      </label>
      <button type="submit" [disabled]="saving()">Create</button>
    </form>
  }
}

@if (certifications().length) {
  <table class="catalog-table">
    <thead>
      <tr>
        <th>Career</th>
        <th>Certification</th>
        <th>Provider</th>
        <th></th>
      </tr>
    </thead>
    <tbody>
      @for (c of certifications(); track c.certification_id) {
        <tr>
          <td>{{ c.career_name }}</td>
          <td>{{ c.certification_name }}</td>
          <td>{{ c.provider ?? '—' }}</td>
          <td>
            @if (canDelete) {
              <button type="button" class="danger" (click)="deleteCertification(c.certification_id)">Delete</button>
            }
          </td>
        </tr>
      }
    </tbody>
  </table>
} @else {
  <p>No certifications in the catalog yet.</p>
}

<h3>Projects Catalog</h3>
@if (canWrite) {
  <button type="button" (click)="toggleProjectForm()">
    {{ showProjectForm() ? 'Cancel' : 'Add Project' }}
  </button>

  @if (showProjectForm()) {
    <form class="catalog-form" [formGroup]="projectForm" (ngSubmit)="createProject()">
      <label>
        Career Name
        <input type="text" formControlName="career_name" />
      </label>
      <label>
        Project Name
        <input type="text" formControlName="project_name" />
      </label>
      <label>
        Description (optional)
        <input type="text" formControlName="description" />
      </label>
      <label>
        Difficulty Level (optional)
        <input type="text" formControlName="difficulty_level" />
      </label>
      <button type="submit" [disabled]="saving()">Create</button>
    </form>
  }
}

@if (projects().length) {
  <table class="catalog-table">
    <thead>
      <tr>
        <th>Career</th>
        <th>Project</th>
        <th>Difficulty</th>
        <th></th>
      </tr>
    </thead>
    <tbody>
      @for (p of projects(); track p.project_id) {
        <tr>
          <td>{{ p.career_name }}</td>
          <td>{{ p.project_name }}</td>
          <td>{{ p.difficulty_level ?? '—' }}</td>
          <td>
            @if (canDelete) {
              <button type="button" class="danger" (click)="deleteProject(p.project_id)">Delete</button>
            }
          </td>
        </tr>
      }
    </tbody>
  </table>
} @else {
  <p>No projects in the catalog yet.</p>
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\counselor-portal\career-criteria\career-criteria.ts
import { Component, inject, signal } from '@angular/core';
import { FormBuilder, ReactiveFormsModule, Validators } from '@angular/forms';

import { AuthService } from '../../../core/auth/auth.service';
import { ROLES } from '../../../core/roles';
import { CareerDefinition, Certification, Project } from '../../../models/catalog.model';
import { CatalogService } from '../../../services/catalog.service';

@Component({
  selector: 'app-career-criteria',
  standalone: true,
  imports: [ReactiveFormsModule],
  templateUrl: './career-criteria.html',
  styleUrl: './career-criteria.css',
})
export class CareerCriteria {
  private readonly fb = inject(FormBuilder);
  private readonly authService = inject(AuthService);
  private readonly catalogService = inject(CatalogService);

  protected readonly canWrite = this.authService.hasAnyRole(ROLES.ADMIN, ROLES.COUNSELOR);
  protected readonly canDelete = this.authService.hasAnyRole(ROLES.ADMIN);

  protected readonly careers = signal<CareerDefinition[]>([]);
  protected readonly certifications = signal<Certification[]>([]);
  protected readonly projects = signal<Project[]>([]);
  protected readonly loading = signal(true);
  protected readonly error = signal<string | null>(null);

  protected readonly showCertForm = signal(false);
  protected readonly showProjectForm = signal(false);
  protected readonly saving = signal(false);

  protected readonly certForm = this.fb.group({
    career_name: ['', Validators.required],
    certification_name: ['', Validators.required],
    provider: [''],
  });

  protected readonly projectForm = this.fb.group({
    career_name: ['', Validators.required],
    project_name: ['', Validators.required],
    description: [''],
    difficulty_level: [''],
  });

  constructor() {
    this.catalogService.getCareers().subscribe({
      next: (careers) => this.careers.set(careers),
      error: () => this.error.set('Failed to load career criteria.'),
    });

    this.catalogService.getCertifications().subscribe({
      next: (certifications) => this.certifications.set(certifications),
      error: () => this.error.set('Failed to load certifications.'),
    });

    this.catalogService.getProjects().subscribe({
      next: (projects) => {
        this.projects.set(projects);
        this.loading.set(false);
      },
      error: () => {
        this.error.set('Failed to load projects.');
        this.loading.set(false);
      },
    });
  }

  protected toggleCertForm(): void {
    this.showCertForm.update((v) => !v);
  }

  protected toggleProjectForm(): void {
    this.showProjectForm.update((v) => !v);
  }

  protected createCertification(): void {
    if (this.certForm.invalid) {
      this.certForm.markAllAsTouched();
      return;
    }
    const raw = this.certForm.getRawValue();
    this.saving.set(true);
    this.catalogService
      .createCertification({
        career_name: raw.career_name ?? '',
        certification_name: raw.certification_name ?? '',
        provider: raw.provider || null,
      })
      .subscribe({
        next: (cert) => {
          this.certifications.update((list) => [...list, cert]);
          this.certForm.reset({ career_name: '', certification_name: '', provider: '' });
          this.showCertForm.set(false);
          this.saving.set(false);
        },
        error: () => {
          this.error.set('Failed to create certification.');
          this.saving.set(false);
        },
      });
  }

  protected deleteCertification(certificationId: number): void {
    this.catalogService.deleteCertification(certificationId).subscribe({
      next: () =>
        this.certifications.update((list) => list.filter((c) => c.certification_id !== certificationId)),
      error: () => this.error.set('Failed to delete certification.'),
    });
  }

  protected createProject(): void {
    if (this.projectForm.invalid) {
      this.projectForm.markAllAsTouched();
      return;
    }
    const raw = this.projectForm.getRawValue();
    this.saving.set(true);
    this.catalogService
      .createProject({
        career_name: raw.career_name ?? '',
        project_name: raw.project_name ?? '',
        description: raw.description || null,
        difficulty_level: raw.difficulty_level || null,
      })
      .subscribe({
        next: (project) => {
          this.projects.update((list) => [...list, project]);
          this.projectForm.reset({ career_name: '', project_name: '', description: '', difficulty_level: '' });
          this.showProjectForm.set(false);
          this.saving.set(false);
        },
        error: () => {
          this.error.set('Failed to create project.');
          this.saving.set(false);
        },
      });
  }

  protected deleteProject(projectId: number): void {
    this.catalogService.deleteProject(projectId).subscribe({
      next: () => this.projects.update((list) => list.filter((p) => p.project_id !== projectId)),
      error: () => this.error.set('Failed to delete project.'),
    });
  }
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\counselor-portal\counselor-shell\counselor-shell.css
.portal-layout {
  display: flex;
  min-height: calc(100vh - 56px);
}

.portal-nav {
  width: 220px;
  flex-shrink: 0;
  display: flex;
  flex-direction: column;
  padding: 1rem 0;
  background: #f9fafb;
  border-right: 1px solid #e5e7eb;
}

.portal-nav a {
  padding: 0.6rem 1.25rem;
  color: #374151;
  text-decoration: none;
  font-size: 0.9rem;
}

.portal-nav a:hover {
  background: #f3f4f6;
}

.portal-nav a.active {
  background: #e0e7ff;
  color: #2563eb;
  font-weight: 600;
}

.portal-content {
  flex: 1;
  padding: 1.5rem;
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\counselor-portal\counselor-shell\counselor-shell.html
<app-header portalTitle="Counselor / Admin Portal" />
<div class="portal-layout">
  <nav class="portal-nav">
    @for (link of navLinks; track link.path) {
      <a [routerLink]="link.path" routerLinkActive="active" [routerLinkActiveOptions]="{ exact: link.path === '' }">
        {{ link.label }}
      </a>
    }
  </nav>
  <main class="portal-content">
    <router-outlet />
  </main>
</div>

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\counselor-portal\counselor-shell\counselor-shell.ts
import { Component } from '@angular/core';
import { RouterLink, RouterLinkActive, RouterOutlet } from '@angular/router';

import { Header } from '../../../shared/layout/header/header';

@Component({
  selector: 'app-counselor-shell',
  standalone: true,
  imports: [RouterOutlet, RouterLink, RouterLinkActive, Header],
  templateUrl: './counselor-shell.html',
  styleUrl: './counselor-shell.css',
})
export class CounselorShell {
  protected readonly navLinks = [
    { path: '', label: 'Dashboard' },
    { path: 'students', label: 'Student Management' },
    { path: 'question-bank', label: 'Question Bank' },
    { path: 'career-criteria', label: 'Career Criteria' },
    { path: 'analytics', label: 'Analytics & Reports' },
  ];
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\counselor-portal\dashboard\dashboard.css
.widget-grid {
  display: flex;
  gap: 1rem;
  flex-wrap: wrap;
  margin: 1rem 0 1.5rem;
}

.widget {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
  min-width: 200px;
  padding: 1rem 1.25rem;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  background: #f9fafb;
}

.widget-value {
  font-size: 1.75rem;
  font-weight: 700;
  color: #2563eb;
}

.widget-label {
  font-size: 0.85rem;
  color: #6b7280;
}

.career-list {
  list-style: none;
  padding: 0;
  max-width: 480px;
}

.career-list li {
  display: flex;
  justify-content: space-between;
  padding: 0.5rem 0;
  border-bottom: 1px solid #e5e7eb;
}

.career-count {
  color: #6b7280;
  font-size: 0.85rem;
}

.quick-links {
  display: flex;
  gap: 1rem;
  margin-top: 1.5rem;
  flex-wrap: wrap;
}

.quick-links a {
  color: #2563eb;
  text-decoration: none;
  font-size: 0.9rem;
}

.quick-links a:hover {
  text-decoration: underline;
}

.error {
  color: #dc2626;
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\counselor-portal\dashboard\dashboard.html
@if (currentUser(); as user) {
  <h2>Welcome, {{ user.full_name }}</h2>
}

@if (loading()) {
  <p>Loading...</p>
} @else if (error()) {
  <p class="error">{{ error() }}</p>
}

@if (summary(); as s) {
  <div class="widget-grid">
    <div class="widget">
      <span class="widget-value">{{ s.active_students }}</span>
      <span class="widget-label">Active Students</span>
    </div>
    <div class="widget">
      <span class="widget-value">{{ s.assessments_completed_today }}</span>
      <span class="widget-label">Assessments Completed Today</span>
    </div>
    <div class="widget">
      <span class="widget-value">{{ s.roadmap_completion_rate.overall_average_completion_percentage }}%</span>
      <span class="widget-label">Avg. Roadmap Completion</span>
    </div>
  </div>

  <h3>Top Recommended Careers</h3>
  @if (s.top_recommended_careers.data.length) {
    <ul class="career-list">
      @for (c of s.top_recommended_careers.data; track c.career_name) {
        <li>
          <span class="career-name">{{ c.career_name }}</span>
          <span class="career-count">{{ c.recommendation_count }} recommendations</span>
        </li>
      }
    </ul>
  } @else {
    <p>No career recommendations recorded yet.</p>
  }

  <div class="quick-links">
    <a routerLink="/counselor/students">Manage Students</a>
    <a routerLink="/counselor/question-bank">Manage Question Bank</a>
    <a routerLink="/counselor/career-criteria">Manage Career Criteria</a>
    <a routerLink="/counselor/analytics">View Full Analytics</a>
  </div>
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\counselor-portal\dashboard\dashboard.ts
import { Component, inject, signal } from '@angular/core';
import { RouterLink } from '@angular/router';

import { AuthService } from '../../../core/auth/auth.service';
import { DashboardSummary } from '../../../models/analytics.model';
import { AnalyticsService } from '../../../services/analytics.service';

@Component({
  selector: 'app-counselor-dashboard',
  standalone: true,
  imports: [RouterLink],
  templateUrl: './dashboard.html',
  styleUrl: './dashboard.css',
})
export class Dashboard {
  private readonly authService = inject(AuthService);
  private readonly analyticsService = inject(AnalyticsService);

  readonly currentUser = this.authService.currentUser;

  protected readonly summary = signal<DashboardSummary | null>(null);
  protected readonly loading = signal(true);
  protected readonly error = signal<string | null>(null);

  constructor() {
    this.analyticsService.getDashboard().subscribe({
      next: (summary) => {
        this.summary.set(summary);
        this.loading.set(false);
      },
      error: () => {
        this.error.set('Failed to load dashboard metrics.');
        this.loading.set(false);
      },
    });
  }
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\counselor-portal\question-bank\question-bank.css
.question-form {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1rem;
  padding: 1rem;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  margin: 1rem 0;
  background: #f9fafb;
  max-width: 700px;
}

.question-form label {
  display: flex;
  flex-direction: column;
  font-size: 0.85rem;
  color: #4b5563;
  gap: 0.25rem;
}

.question-form label.full-width {
  grid-column: 1 / -1;
}

.question-form input {
  padding: 0.4rem;
  border: 1px solid #d1d5db;
  border-radius: 4px;
}

.edit-actions {
  grid-column: 1 / -1;
  display: flex;
  gap: 0.75rem;
}

.question-list {
  list-style: none;
  padding: 0;
  max-width: 700px;
}

.question-list > li {
  padding: 1rem;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  margin-bottom: 1rem;
}

.question-header {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  margin-bottom: 0.5rem;
}

.category-badge {
  background: #e0e7ff;
  color: #2563eb;
  border-radius: 4px;
  padding: 0.15rem 0.5rem;
  font-size: 0.75rem;
  font-weight: 600;
}

.version-badge {
  color: #6b7280;
  font-size: 0.75rem;
}

.question-text {
  font-weight: 500;
  margin: 0.25rem 0 0.5rem;
}

.option-list {
  list-style: none;
  padding: 0;
  margin: 0;
  font-size: 0.85rem;
  color: #4b5563;
}

.error {
  color: #dc2626;
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\counselor-portal\question-bank\question-bank.html
<h2>Question Bank Management</h2>

@if (loading()) {
  <p>Loading...</p>
} @else if (error()) {
  <p class="error">{{ error() }}</p>
}

@if (canWrite) {
  <button type="button" (click)="toggleCreateForm()">
    {{ showCreateForm() ? 'Cancel' : 'Add Question' }}
  </button>

  @if (showCreateForm()) {
    <form class="question-form" [formGroup]="createForm" (ngSubmit)="createQuestion()">
      <label>
        Category
        <input type="text" formControlName="category" />
      </label>
      <label class="full-width">
        Question Text
        <input type="text" formControlName="question_text" />
      </label>
      <label>
        Option A
        <input type="text" formControlName="option_a" />
      </label>
      <label>
        Option B
        <input type="text" formControlName="option_b" />
      </label>
      <label>
        Option C (optional)
        <input type="text" formControlName="option_c" />
      </label>
      <label>
        Option D (optional)
        <input type="text" formControlName="option_d" />
      </label>
      <button type="submit" [disabled]="saving()">Create</button>
    </form>
  }
}

@if (!loading()) {
  @if (questions().length) {
    <ul class="question-list">
      @for (q of questions(); track q.question_id) {
        <li>
          @if (editingId() === q.question_id) {
            <form class="question-form" [formGroup]="editForm" (ngSubmit)="saveEdit(q.question_id)">
              <label>
                Category
                <input type="text" formControlName="category" />
              </label>
              <label class="full-width">
                Question Text
                <input type="text" formControlName="question_text" />
              </label>
              <label>
                Option A
                <input type="text" formControlName="option_a" />
              </label>
              <label>
                Option B
                <input type="text" formControlName="option_b" />
              </label>
              <label>
                Option C (optional)
                <input type="text" formControlName="option_c" />
              </label>
              <label>
                Option D (optional)
                <input type="text" formControlName="option_d" />
              </label>
              <div class="edit-actions">
                <button type="submit" [disabled]="saving()">Save</button>
                <button type="button" (click)="cancelEdit()">Cancel</button>
              </div>
            </form>
          } @else {
            <div class="question-header">
              <span class="category-badge">{{ q.category }}</span>
              <span class="version-badge">v{{ q.version }}</span>
              @if (canWrite) {
                <button type="button" (click)="startEdit(q)">Edit</button>
              }
            </div>
            <p class="question-text">{{ q.question_text }}</p>
            <ul class="option-list">
              @for (entry of optionEntries(q.options); track entry[0]) {
                <li><strong>{{ entry[0] }}:</strong> {{ entry[1] }}</li>
              }
            </ul>
          }
        </li>
      }
    </ul>
  } @else {
    <p>No questions in the bank yet.</p>
  }
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\counselor-portal\question-bank\question-bank.ts
import { Component, inject, signal } from '@angular/core';
import { FormBuilder, ReactiveFormsModule, Validators } from '@angular/forms';

import { AuthService } from '../../../core/auth/auth.service';
import { ROLES } from '../../../core/roles';
import { DiscoveryQuestion } from '../../../models/discovery.model';
import { DiscoveryService } from '../../../services/discovery.service';

@Component({
  selector: 'app-question-bank',
  standalone: true,
  imports: [ReactiveFormsModule],
  templateUrl: './question-bank.html',
  styleUrl: './question-bank.css',
})
export class QuestionBank {
  private readonly fb = inject(FormBuilder);
  private readonly authService = inject(AuthService);
  private readonly discoveryService = inject(DiscoveryService);

  protected readonly canWrite = this.authService.hasAnyRole(ROLES.ADMIN, ROLES.COUNSELOR);

  protected readonly questions = signal<DiscoveryQuestion[]>([]);
  protected readonly loading = signal(true);
  protected readonly error = signal<string | null>(null);
  protected readonly showCreateForm = signal(false);
  protected readonly saving = signal(false);
  protected readonly editingId = signal<number | null>(null);

  protected readonly createForm = this.fb.group({
    category: ['', [Validators.required, Validators.maxLength(100)]],
    question_text: ['', [Validators.required, Validators.maxLength(500)]],
    option_a: ['', Validators.required],
    option_b: ['', Validators.required],
    option_c: [''],
    option_d: [''],
  });

  protected readonly editForm = this.fb.group({
    category: ['', [Validators.required, Validators.maxLength(100)]],
    question_text: ['', [Validators.required, Validators.maxLength(500)]],
    option_a: ['', Validators.required],
    option_b: ['', Validators.required],
    option_c: [''],
    option_d: [''],
  });

  constructor() {
    this.loadQuestions();
  }

  private loadQuestions(): void {
    this.loading.set(true);
    this.discoveryService.listQuestions().subscribe({
      next: (questions) => {
        this.questions.set(questions);
        this.loading.set(false);
      },
      error: () => {
        this.error.set('Failed to load the question bank.');
        this.loading.set(false);
      },
    });
  }

  private buildOptions(raw: {
    option_a: string | null;
    option_b: string | null;
    option_c: string | null;
    option_d: string | null;
  }): Record<string, string> {
    const options: Record<string, string> = {};
    if (raw.option_a) options['A'] = raw.option_a;
    if (raw.option_b) options['B'] = raw.option_b;
    if (raw.option_c) options['C'] = raw.option_c;
    if (raw.option_d) options['D'] = raw.option_d;
    return options;
  }

  protected toggleCreateForm(): void {
    this.showCreateForm.update((v) => !v);
  }

  protected createQuestion(): void {
    if (this.createForm.invalid) {
      this.createForm.markAllAsTouched();
      return;
    }

    const raw = this.createForm.getRawValue();
    this.saving.set(true);
    this.discoveryService
      .createQuestion({
        category: raw.category ?? '',
        question_text: raw.question_text ?? '',
        options: this.buildOptions(raw),
      })
      .subscribe({
        next: (question) => {
          this.questions.update((list) => [...list, question]);
          this.createForm.reset({ category: '', question_text: '', option_a: '', option_b: '', option_c: '', option_d: '' });
          this.showCreateForm.set(false);
          this.saving.set(false);
        },
        error: () => {
          this.error.set('Failed to create the question.');
          this.saving.set(false);
        },
      });
  }

  protected startEdit(question: DiscoveryQuestion): void {
    this.editingId.set(question.question_id);
    this.editForm.reset({
      category: question.category,
      question_text: question.question_text,
      option_a: question.options['A'] ?? '',
      option_b: question.options['B'] ?? '',
      option_c: question.options['C'] ?? '',
      option_d: question.options['D'] ?? '',
    });
  }

  protected cancelEdit(): void {
    this.editingId.set(null);
  }

  protected saveEdit(questionId: number): void {
    if (this.editForm.invalid) {
      this.editForm.markAllAsTouched();
      return;
    }

    const raw = this.editForm.getRawValue();
    this.saving.set(true);
    this.discoveryService
      .updateQuestion(questionId, {
        category: raw.category ?? '',
        question_text: raw.question_text ?? '',
        options: this.buildOptions(raw),
      })
      .subscribe({
        next: (updated) => {
          this.questions.update((list) => list.map((q) => (q.question_id === questionId ? updated : q)));
          this.editingId.set(null);
          this.saving.set(false);
        },
        error: () => {
          this.error.set('Failed to update the question.');
          this.saving.set(false);
        },
      });
  }

  protected optionEntries(options: Record<string, string>): [string, string][] {
    return Object.entries(options);
  }
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\counselor-portal\student-management\student-detail\student-detail.css
.profile-grid {
  display: grid;
  grid-template-columns: 140px 1fr;
  row-gap: 0.5rem;
  max-width: 480px;
  margin-bottom: 1.5rem;
}

.profile-grid dt {
  font-weight: 600;
  color: #4b5563;
}

.profile-grid dd {
  margin: 0;
}

.data-table {
  width: 100%;
  border-collapse: collapse;
  max-width: 700px;
  margin-bottom: 1.5rem;
}

.data-table th,
.data-table td {
  text-align: left;
  padding: 0.5rem 0.75rem;
  border-bottom: 1px solid #e5e7eb;
}

.skill-list,
.career-list {
  list-style: none;
  padding: 0;
  max-width: 480px;
}

.skill-list li {
  display: grid;
  grid-template-columns: 140px 1fr 48px;
  align-items: center;
  gap: 0.5rem;
  margin-bottom: 0.5rem;
}

.bar {
  height: 8px;
  background: #e5e7eb;
  border-radius: 4px;
  overflow: hidden;
}

.bar-fill {
  height: 100%;
  background: #2563eb;
}

.skill-score {
  font-size: 0.85rem;
  text-align: right;
}

.career-list li {
  display: flex;
  justify-content: space-between;
  padding: 0.4rem 0;
  border-bottom: 1px solid #e5e7eb;
}

.error {
  color: #dc2626;
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\counselor-portal\student-management\student-detail\student-detail.html
<a routerLink="/counselor/students">&larr; Back to Student Management</a>

@if (loading()) {
  <p>Loading...</p>
} @else if (error()) {
  <p class="error">{{ error() }}</p>
}

@if (student(); as s) {
  <h2>{{ s.name }}</h2>
  <dl class="profile-grid">
    <dt>Age</dt>
    <dd>{{ s.age }}</dd>
    <dt>Education</dt>
    <dd>{{ s.education }}</dd>
    <dt>Location</dt>
    <dd>{{ s.location ?? '—' }}</dd>
    <dt>Board</dt>
    <dd>{{ s.board ?? '—' }}</dd>
    <dt>Joined</dt>
    <dd>{{ s.created_date | date: 'mediumDate' }}</dd>
  </dl>

  <h3>Academic History</h3>
  @if (history()?.academic_history?.length) {
    <table class="data-table">
      <thead>
        <tr>
          <th>Subject</th>
          <th>Score</th>
          <th>Attendance</th>
          <th>Recorded</th>
        </tr>
      </thead>
      <tbody>
        @for (h of history()!.academic_history; track h.history_id) {
          <tr>
            <td>{{ h.subject }}</td>
            <td>{{ h.score }}</td>
            <td>{{ h.attendance ?? '—' }}</td>
            <td>{{ h.recorded_date | date: 'mediumDate' }}</td>
          </tr>
        }
      </tbody>
    </table>
  } @else {
    <p>No academic history recorded.</p>
  }

  <h3>Skills</h3>
  @if (skills().length) {
    <ul class="skill-list">
      @for (skill of skills(); track skill.id) {
        <li>
          <span class="skill-name">{{ skill.skill_name }}</span>
          <div class="bar">
            <div class="bar-fill" [style.width.%]="skill.proficiency_score"></div>
          </div>
          <span class="skill-score">{{ skill.proficiency_score }}%</span>
        </li>
      }
    </ul>
  } @else {
    <p>No skills assessed yet.</p>
  }

  <h3>Career Recommendations</h3>
  @if (recommendations().length) {
    <ul class="career-list">
      @for (r of recommendations(); track r.recommendation_id) {
        <li>
          <span class="career-name">{{ r.career_name }}</span>
          <span class="career-score">Match: {{ r.match_score }}%</span>
        </li>
      }
    </ul>
  } @else {
    <p>No career recommendations yet.</p>
  }
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\counselor-portal\student-management\student-detail\student-detail.ts
import { Component, inject, signal } from '@angular/core';
import { ActivatedRoute, RouterLink } from '@angular/router';
import { DatePipe } from '@angular/common';

import { StudentHistory } from '../../../../models/history.model';
import { CareerRecommendation } from '../../../../models/recommendation.model';
import { Student } from '../../../../models/student.model';
import { StudentSkill } from '../../../../models/skill.model';
import { RecommendationService } from '../../../../services/recommendation.service';
import { StudentService } from '../../../../services/student.service';
import { SkillService } from '../../../../services/skill.service';

@Component({
  selector: 'app-student-detail',
  standalone: true,
  imports: [RouterLink, DatePipe],
  templateUrl: './student-detail.html',
  styleUrl: './student-detail.css',
})
export class StudentDetail {
  private readonly route = inject(ActivatedRoute);
  private readonly studentService = inject(StudentService);
  private readonly skillService = inject(SkillService);
  private readonly recommendationService = inject(RecommendationService);

  protected readonly student = signal<Student | null>(null);
  protected readonly history = signal<StudentHistory | null>(null);
  protected readonly skills = signal<StudentSkill[]>([]);
  protected readonly recommendations = signal<CareerRecommendation[]>([]);
  protected readonly loading = signal(true);
  protected readonly error = signal<string | null>(null);

  constructor() {
    const studentId = Number(this.route.snapshot.paramMap.get('id'));
    if (!studentId) {
      this.error.set('Invalid student id.');
      this.loading.set(false);
      return;
    }

    this.studentService.getStudent(studentId).subscribe({
      next: (student) => this.student.set(student),
      error: () => this.error.set('Failed to load the student profile.'),
    });

    this.studentService.getHistory(studentId).subscribe({
      next: (history) => this.history.set(history),
      error: () => this.error.set('Failed to load the student history.'),
    });

    this.skillService.getSkills(studentId).subscribe({
      next: (skills) => this.skills.set(skills),
      error: () => this.error.set('Failed to load the student skills.'),
    });

    this.recommendationService.getForStudent(studentId).subscribe({
      next: (recommendations) => {
        this.recommendations.set(recommendations);
        this.loading.set(false);
      },
      error: () => {
        this.error.set('Failed to load the student recommendations.');
        this.loading.set(false);
      },
    });
  }
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\counselor-portal\student-management\student-management.css
.toolbar {
  display: flex;
  gap: 1rem;
  align-items: center;
  margin: 1rem 0;
}

.toolbar input {
  flex: 1;
  max-width: 320px;
  padding: 0.5rem;
  border: 1px solid #d1d5db;
  border-radius: 4px;
}

.create-form {
  display: flex;
  flex-wrap: wrap;
  gap: 1rem;
  padding: 1rem;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  margin-bottom: 1.5rem;
  background: #f9fafb;
}

.create-form label {
  display: flex;
  flex-direction: column;
  font-size: 0.85rem;
  color: #4b5563;
  gap: 0.25rem;
}

.create-form input {
  padding: 0.4rem;
  border: 1px solid #d1d5db;
  border-radius: 4px;
}

.student-table {
  width: 100%;
  border-collapse: collapse;
  max-width: 800px;
}

.student-table th,
.student-table td {
  text-align: left;
  padding: 0.5rem 0.75rem;
  border-bottom: 1px solid #e5e7eb;
}

.actions {
  display: flex;
  gap: 0.75rem;
  align-items: center;
}

.actions a {
  color: #2563eb;
  text-decoration: none;
}

button.danger {
  color: #dc2626;
  background: none;
  border: 1px solid #dc2626;
  border-radius: 4px;
  padding: 0.2rem 0.5rem;
  cursor: pointer;
}

.error {
  color: #dc2626;
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\counselor-portal\student-management\student-management.html
<h2>Student Management</h2>

@if (loading()) {
  <p>Loading...</p>
} @else if (error()) {
  <p class="error">{{ error() }}</p>
}

<div class="toolbar">
  <input
    type="text"
    placeholder="Search by name or education..."
    [value]="searchTerm()"
    (input)="searchTerm.set($any($event.target).value)"
  />
  @if (canWrite) {
    <button type="button" (click)="toggleCreateForm()">
      {{ showCreateForm() ? 'Cancel' : 'Add Student' }}
    </button>
  }
</div>

@if (showCreateForm()) {
  <form class="create-form" [formGroup]="form" (ngSubmit)="createStudent()">
    <label>
      Name
      <input type="text" formControlName="name" />
    </label>
    <label>
      Age
      <input type="number" formControlName="age" />
    </label>
    <label>
      Education
      <input type="text" formControlName="education" />
    </label>
    <label>
      Location
      <input type="text" formControlName="location" />
    </label>
    <label>
      Board
      <input type="text" formControlName="board" />
    </label>
    <button type="submit" [disabled]="creating()">Create</button>
  </form>
}

@if (!loading()) {
  @if (filteredStudents().length) {
    <table class="student-table">
      <thead>
        <tr>
          <th>Name</th>
          <th>Age</th>
          <th>Education</th>
          <th>Location</th>
          <th></th>
        </tr>
      </thead>
      <tbody>
        @for (student of filteredStudents(); track student.student_id) {
          <tr>
            <td>{{ student.name }}</td>
            <td>{{ student.age }}</td>
            <td>{{ student.education }}</td>
            <td>{{ student.location ?? '—' }}</td>
            <td class="actions">
              <a [routerLink]="[student.student_id]">View</a>
              @if (canDelete) {
                <button type="button" class="danger" (click)="deleteStudent(student.student_id)">Delete</button>
              }
            </td>
          </tr>
        }
      </tbody>
    </table>
  } @else {
    <p>No students found.</p>
  }
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\counselor-portal\student-management\student-management.ts
import { Component, inject, signal } from '@angular/core';
import { FormBuilder, ReactiveFormsModule, Validators } from '@angular/forms';
import { RouterLink } from '@angular/router';

import { AuthService } from '../../../core/auth/auth.service';
import { ROLES } from '../../../core/roles';
import { Student } from '../../../models/student.model';
import { StudentService } from '../../../services/student.service';

@Component({
  selector: 'app-student-management',
  standalone: true,
  imports: [ReactiveFormsModule, RouterLink],
  templateUrl: './student-management.html',
  styleUrl: './student-management.css',
})
export class StudentManagement {
  private readonly fb = inject(FormBuilder);
  private readonly authService = inject(AuthService);
  private readonly studentService = inject(StudentService);

  protected readonly canWrite = this.authService.hasAnyRole(ROLES.ADMIN, ROLES.COUNSELOR);
  protected readonly canDelete = this.authService.hasAnyRole(ROLES.ADMIN);

  protected readonly students = signal<Student[]>([]);
  protected readonly loading = signal(true);
  protected readonly error = signal<string | null>(null);
  protected readonly searchTerm = signal('');
  protected readonly showCreateForm = signal(false);
  protected readonly creating = signal(false);

  protected readonly form = this.fb.group({
    name: ['', [Validators.required, Validators.maxLength(150)]],
    age: [18, [Validators.required, Validators.min(10), Validators.max(100)]],
    education: ['', [Validators.required, Validators.maxLength(150)]],
    location: [''],
    board: [''],
  });

  constructor() {
    this.loadStudents();
  }

  private loadStudents(): void {
    this.loading.set(true);
    this.studentService.listStudents(0, 200).subscribe({
      next: (students) => {
        this.students.set(students);
        this.loading.set(false);
      },
      error: () => {
        this.error.set('Failed to load students.');
        this.loading.set(false);
      },
    });
  }

  protected filteredStudents(): Student[] {
    const term = this.searchTerm().trim().toLowerCase();
    if (!term) {
      return this.students();
    }
    return this.students().filter(
      (s) => s.name.toLowerCase().includes(term) || s.education.toLowerCase().includes(term),
    );
  }

  protected toggleCreateForm(): void {
    this.showCreateForm.update((v) => !v);
  }

  protected createStudent(): void {
    if (this.form.invalid) {
      this.form.markAllAsTouched();
      return;
    }

    const raw = this.form.getRawValue();
    this.creating.set(true);
    this.studentService
      .createStudent({
        name: raw.name ?? '',
        age: raw.age ?? 18,
        education: raw.education ?? '',
        location: raw.location || null,
        board: raw.board || null,
      })
      .subscribe({
        next: (student) => {
          this.students.update((list) => [...list, student]);
          this.form.reset({ name: '', age: 18, education: '', location: '', board: '' });
          this.showCreateForm.set(false);
          this.creating.set(false);
        },
        error: () => {
          this.error.set('Failed to create student.');
          this.creating.set(false);
        },
      });
  }

  protected deleteStudent(studentId: number): void {
    this.studentService.deleteStudent(studentId).subscribe({
      next: () => this.students.update((list) => list.filter((s) => s.student_id !== studentId)),
      error: () => this.error.set('Failed to delete student.'),
    });
  }
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\student-portal\academic-history\academic-history.css
table {
  border-collapse: collapse;
  width: 100%;
  max-width: 720px;
  margin-bottom: 1.5rem;
}

th,
td {
  text-align: left;
  padding: 0.5rem 0.75rem;
  border-bottom: 1px solid #e5e7eb;
}

th {
  background: #f9fafb;
  font-size: 0.85rem;
  color: #4b5563;
}

.error {
  color: #dc2626;
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\student-portal\academic-history\academic-history.html
<h2>Academic &amp; Learning History</h2>

@if (loading()) {
  <p>Loading...</p>
} @else if (error()) {
  <p class="error">{{ error() }}</p>
} @else {
  @if (history(); as h) {
    <h3>Academic History</h3>
    @if (h.academic_history.length) {
      <table>
        <thead>
          <tr>
            <th>Subject</th>
            <th>Score</th>
            <th>Attendance</th>
            <th>Behavior</th>
            <th>Recorded</th>
          </tr>
        </thead>
        <tbody>
          @for (row of h.academic_history; track row.history_id) {
            <tr>
              <td>{{ row.subject }}</td>
              <td>{{ row.score }}</td>
              <td>{{ row.attendance ?? '—' }}</td>
              <td>{{ row.learning_behavior ?? '—' }}</td>
              <td>{{ row.recorded_date | date: 'mediumDate' }}</td>
            </tr>
          }
        </tbody>
      </table>
    } @else {
      <p>No academic history recorded yet.</p>
    }

    <h3>Learning Platforms</h3>
    @if (h.learning_platform_data.length) {
      <table>
        <thead>
          <tr>
            <th>Platform</th>
            <th>Courses Completed</th>
            <th>Hours Spent</th>
            <th>Skill Area</th>
            <th>Recorded</th>
          </tr>
        </thead>
        <tbody>
          @for (row of h.learning_platform_data; track row.record_id) {
            <tr>
              <td>{{ row.platform }}</td>
              <td>{{ row.courses_completed }}</td>
              <td>{{ row.hours_spent }}</td>
              <td>{{ row.skill_area ?? '—' }}</td>
              <td>{{ row.recorded_date | date: 'mediumDate' }}</td>
            </tr>
          }
        </tbody>
      </table>
    } @else {
      <p>No learning platform data recorded yet.</p>
    }
  }
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\student-portal\academic-history\academic-history.ts
import { DatePipe } from '@angular/common';
import { Component, inject, signal } from '@angular/core';

import { AuthService } from '../../../core/auth/auth.service';
import { StudentHistory } from '../../../models/history.model';
import { StudentService } from '../../../services/student.service';

@Component({
  selector: 'app-academic-history',
  standalone: true,
  imports: [DatePipe],
  templateUrl: './academic-history.html',
  styleUrl: './academic-history.css',
})
export class AcademicHistory {
  private readonly authService = inject(AuthService);
  private readonly studentService = inject(StudentService);

  protected readonly history = signal<StudentHistory | null>(null);
  protected readonly loading = signal(true);
  protected readonly error = signal<string | null>(null);

  constructor() {
    const studentId = this.authService.currentUser()?.student_id;
    if (!studentId) {
      this.error.set('No student profile is linked to your account.');
      this.loading.set(false);
      return;
    }

    this.studentService.getHistory(studentId).subscribe({
      next: (history) => {
        this.history.set(history);
        this.loading.set(false);
      },
      error: () => {
        this.error.set('Failed to load your academic history.');
        this.loading.set(false);
      },
    });
  }
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\student-portal\dashboard\dashboard.ts
import { Component, inject } from '@angular/core';
import { RouterLink } from '@angular/router';

import { AuthService } from '../../../core/auth/auth.service';

@Component({
  selector: 'app-student-dashboard',
  standalone: true,
  imports: [RouterLink],
  template: `
    @if (currentUser(); as user) {
      <h2>Welcome, {{ user.full_name }}</h2>
    }
    <p>Jump back into your career journey:</p>
    <ul>
      <li><a routerLink="/student/recommendations">Career Recommendations</a></li>
      <li><a routerLink="/student/roadmap">Learning Roadmap</a></li>
      <li><a routerLink="/student/mentor-chat">Mentor Chat</a></li>
      <li><a routerLink="/student/progress">Progress Tracker</a></li>
    </ul>
  `,
})
export class Dashboard {
  private readonly authService = inject(AuthService);
  readonly currentUser = this.authService.currentUser;
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\student-portal\discovery\discovery.css
.chat-log {
  max-width: 560px;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.bubble {
  padding: 0.6rem 0.9rem;
  border-radius: 12px;
  max-width: 80%;
}

.bubble.bot {
  background: #f3f4f6;
  align-self: flex-start;
}

.bubble.user {
  background: #2563eb;
  color: #fff;
  align-self: flex-end;
}

.options {
  display: flex;
  flex-wrap: wrap;
  gap: 0.5rem;
  margin-bottom: 1rem;
}

.options button {
  padding: 0.5rem 0.9rem;
  border: 1px solid #2563eb;
  color: #2563eb;
  background: #fff;
  border-radius: 6px;
  cursor: pointer;
}

.options button:hover {
  background: #eff6ff;
}

.options button:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.error {
  color: #dc2626;
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\student-portal\discovery\discovery.html
<h2>Discovery Questions</h2>

@if (error()) {
  <p class="error">{{ error() }}</p>
}

<div class="chat-log">
  @for (entry of history(); track entry.questionText) {
    <div class="bubble bot">{{ entry.questionText }}</div>
    <div class="bubble user">{{ entry.answerLabel }}</div>
  }

  @if (loading()) {
    <p>Loading...</p>
  } @else if (complete()) {
    <div class="bubble bot">You've completed the discovery questionnaire. Thanks for sharing!</div>
  } @else {
    @if (currentQuestion(); as q) {
      <div class="bubble bot">{{ q.question_text }}</div>
      <div class="options">
        @for (entry of optionEntries(q); track entry[0]) {
          <button type="button" [disabled]="submitting()" (click)="selectOption(entry[0], entry[1])">
            {{ entry[1] }}
          </button>
        }
      </div>
    }
  }
</div>

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\student-portal\discovery\discovery.ts
import { Component, inject, signal } from '@angular/core';

import { AuthService } from '../../../core/auth/auth.service';
import { DiscoveryQuestion } from '../../../models/discovery.model';
import { DiscoveryService } from '../../../services/discovery.service';

interface ChatEntry {
  questionText: string;
  answerLabel: string;
}

@Component({
  selector: 'app-discovery',
  standalone: true,
  templateUrl: './discovery.html',
  styleUrl: './discovery.css',
})
export class Discovery {
  private readonly authService = inject(AuthService);
  private readonly discoveryService = inject(DiscoveryService);
  private readonly studentId = this.authService.currentUser()?.student_id ?? null;

  protected readonly history = signal<ChatEntry[]>([]);
  protected readonly currentQuestion = signal<DiscoveryQuestion | null>(null);
  protected readonly loading = signal(true);
  protected readonly submitting = signal(false);
  protected readonly complete = signal(false);
  protected readonly error = signal<string | null>(null);

  constructor() {
    if (!this.studentId) {
      this.error.set('No student profile is linked to your account.');
      this.loading.set(false);
      return;
    }
    this.loadNextQuestion();
  }

  protected optionEntries(question: DiscoveryQuestion): Array<[string, string]> {
    return Object.entries(question.options);
  }

  protected selectOption(option: string, label: string): void {
    const question = this.currentQuestion();
    if (!this.studentId || !question || this.submitting()) {
      return;
    }

    this.submitting.set(true);
    this.discoveryService
      .submitAnswer({ student_id: this.studentId, question_id: question.question_id, selected_option: option })
      .subscribe({
        next: () => {
          this.history.update((entries) => [
            ...entries,
            { questionText: question.question_text, answerLabel: label },
          ]);
          this.submitting.set(false);
          this.loadNextQuestion();
        },
        error: () => {
          this.error.set('Failed to submit your answer.');
          this.submitting.set(false);
        },
      });
  }

  private loadNextQuestion(): void {
    if (!this.studentId) {
      return;
    }
    this.loading.set(true);
    this.discoveryService.getNextQuestion(this.studentId).subscribe({
      next: (question) => {
        this.currentQuestion.set(question);
        this.complete.set(!question);
        this.loading.set(false);
      },
      error: () => {
        this.error.set('Failed to load the next question.');
        this.loading.set(false);
      },
    });
  }
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\student-portal\gap-analysis\gap-analysis.css
.career-picker {
  display: flex;
  gap: 0.5rem;
  margin-bottom: 1rem;
  flex-wrap: wrap;
}

.career-picker button {
  padding: 0.4rem 0.8rem;
  border: 1px solid #d1d5db;
  background: #fff;
  border-radius: 6px;
  cursor: pointer;
}

.career-picker button.active {
  background: #2563eb;
  color: #fff;
  border-color: #2563eb;
}

.gap-columns {
  display: flex;
  gap: 2rem;
}

.gap-columns ul {
  padding-left: 1.2rem;
}

.matched {
  color: #16a34a;
}

.missing {
  color: #dc2626;
}

.error {
  color: #dc2626;
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\student-portal\gap-analysis\gap-analysis.html
<h2>Skill Gap Analysis</h2>

@if (error()) {
  <p class="error">{{ error() }}</p>
}

@if (recommendations().length > 1) {
  <div class="career-picker">
    @for (rec of recommendations(); track rec.recommendation_id) {
      <button
        type="button"
        [class.active]="selectedCareer() === rec.career_name"
        (click)="loadGaps(rec)"
      >
        {{ rec.career_name }}
      </button>
    }
  </div>
}

@if (loading()) {
  <p>Loading...</p>
} @else {
  @if (result(); as r) {
    <h3>{{ selectedCareer() }}</h3>
    <div class="gap-columns">
      <div>
        <h4>Matched Skills</h4>
        @if (r.matched.length) {
          <ul>
            @for (skill of r.matched; track skill) {
              <li class="matched">{{ skill }}</li>
            }
          </ul>
        } @else {
          <p>None yet.</p>
        }
      </div>
      <div>
        <h4>Missing Skills</h4>
        @if (r.missing.length) {
          <ul>
            @for (skill of r.missing; track skill) {
              <li class="missing">{{ skill }}</li>
            }
          </ul>
        } @else {
          <p>No gaps found.</p>
        }
      </div>
    </div>
  } @else if (!recommendations().length) {
    <p>No career recommendations yet, so there's nothing to analyze.</p>
  }
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\student-portal\gap-analysis\gap-analysis.ts
import { Component, inject, signal } from '@angular/core';
import { ActivatedRoute } from '@angular/router';

import { AuthService } from '../../../core/auth/auth.service';
import { GapAnalysisResponse } from '../../../models/gap-analysis.model';
import { CareerRecommendation } from '../../../models/recommendation.model';
import { GapAnalysisService } from '../../../services/gap-analysis.service';
import { RecommendationService } from '../../../services/recommendation.service';

@Component({
  selector: 'app-gap-analysis',
  standalone: true,
  templateUrl: './gap-analysis.html',
  styleUrl: './gap-analysis.css',
})
export class GapAnalysis {
  private readonly authService = inject(AuthService);
  private readonly recommendationService = inject(RecommendationService);
  private readonly gapAnalysisService = inject(GapAnalysisService);
  private readonly route = inject(ActivatedRoute);

  protected readonly recommendations = signal<CareerRecommendation[]>([]);
  protected readonly result = signal<GapAnalysisResponse | null>(null);
  protected readonly selectedCareer = signal<string | null>(null);
  protected readonly loading = signal(true);
  protected readonly error = signal<string | null>(null);

  constructor() {
    const studentId = this.authService.currentUser()?.student_id;
    if (!studentId) {
      this.error.set('No student profile is linked to your account.');
      this.loading.set(false);
      return;
    }

    this.recommendationService.getForStudent(studentId).subscribe({
      next: (recommendations) => {
        this.recommendations.set(recommendations);
        const requested = Number(this.route.snapshot.queryParamMap.get('recommendationId'));
        const target = requested
          ? recommendations.find((r) => r.recommendation_id === requested)
          : recommendations[0];
        if (target) {
          this.loadGaps(target);
        } else {
          this.loading.set(false);
        }
      },
      error: () => {
        this.error.set('Failed to load your career recommendations.');
        this.loading.set(false);
      },
    });
  }

  protected loadGaps(recommendation: CareerRecommendation): void {
    this.selectedCareer.set(recommendation.career_name);
    this.loading.set(true);
    this.gapAnalysisService.getByRecommendation(recommendation.recommendation_id).subscribe({
      next: (result) => {
        this.result.set(result);
        this.loading.set(false);
      },
      error: () => {
        this.error.set('Failed to load the skill gap analysis.');
        this.loading.set(false);
      },
    });
  }
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\student-portal\mentor-chat\mentor-chat.css
.chat-log {
  max-width: 560px;
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  margin-bottom: 1rem;
}

.bubble {
  padding: 0.6rem 0.9rem;
  border-radius: 12px;
  max-width: 80%;
}

.bubble.bot {
  background: #f3f4f6;
  align-self: flex-start;
}

.bubble.user {
  background: #2563eb;
  color: #fff;
  align-self: flex-end;
}

.related-career {
  font-size: 0.75rem;
  margin-top: 0.3rem;
  color: #4b5563;
}

.chat-input {
  display: flex;
  gap: 0.5rem;
  max-width: 560px;
}

.chat-input textarea {
  flex: 1;
  padding: 0.5rem;
  border: 1px solid #d1d5db;
  border-radius: 4px;
  resize: vertical;
}

.chat-input button {
  padding: 0.5rem 1rem;
  background: #2563eb;
  color: #fff;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.chat-input button:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.error {
  color: #dc2626;
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\student-portal\mentor-chat\mentor-chat.html
<h2>Mentor Chat</h2>

@if (error()) {
  <p class="error">{{ error() }}</p>
}

@if (loading()) {
  <p>Loading...</p>
} @else {
  <div class="chat-log">
    @for (entry of conversations(); track entry.conversation_id) {
      <div class="bubble user">{{ entry.message }}</div>
      @if (entry.reply) {
        <div class="bubble bot">
          {{ entry.reply }}
          @if (entry.related_career) {
            <div class="related-career">Related career: {{ entry.related_career }}</div>
          }
        </div>
      }
    }
  </div>

  <form [formGroup]="form" (ngSubmit)="send()" class="chat-input">
    <textarea formControlName="message" rows="2" placeholder="Ask your mentor a question..."></textarea>
    <button type="submit" [disabled]="sending()">{{ sending() ? 'Sending...' : 'Send' }}</button>
  </form>
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\student-portal\mentor-chat\mentor-chat.ts
import { Component, inject, signal } from '@angular/core';
import { FormBuilder, ReactiveFormsModule, Validators } from '@angular/forms';

import { AuthService } from '../../../core/auth/auth.service';
import { MentorConversation } from '../../../models/mentor.model';
import { MentorService } from '../../../services/mentor.service';

@Component({
  selector: 'app-mentor-chat',
  standalone: true,
  imports: [ReactiveFormsModule],
  templateUrl: './mentor-chat.html',
  styleUrl: './mentor-chat.css',
})
export class MentorChat {
  private readonly fb = inject(FormBuilder);
  private readonly authService = inject(AuthService);
  private readonly mentorService = inject(MentorService);
  private readonly studentId = this.authService.currentUser()?.student_id ?? null;

  protected readonly form = this.fb.group({
    message: ['', [Validators.required, Validators.maxLength(2000)]],
  });

  protected readonly conversations = signal<MentorConversation[]>([]);
  protected readonly loading = signal(true);
  protected readonly sending = signal(false);
  protected readonly error = signal<string | null>(null);

  constructor() {
    if (!this.studentId) {
      this.error.set('No student profile is linked to your account.');
      this.loading.set(false);
      return;
    }

    this.mentorService.getConversations(this.studentId).subscribe({
      next: (conversations) => {
        this.conversations.set(conversations);
        this.loading.set(false);
      },
      error: () => {
        this.error.set('Failed to load your mentor conversations.');
        this.loading.set(false);
      },
    });
  }

  protected send(): void {
    if (this.form.invalid || !this.studentId) {
      this.form.markAllAsTouched();
      return;
    }

    const message = this.form.getRawValue().message ?? '';
    this.sending.set(true);
    this.mentorService.chat({ studentId: this.studentId, message }).subscribe({
      next: (response) => {
        this.conversations.update((entries) => [
          ...entries,
          {
            conversation_id: -entries.length - 1,
            student_id: this.studentId!,
            message,
            reply: response.reply,
            related_career: response.relatedCareer,
            created_date: new Date().toISOString(),
          },
        ]);
        this.form.reset({ message: '' });
        this.sending.set(false);
      },
      error: () => {
        this.error.set('Failed to send your message.');
        this.sending.set(false);
      },
    });
  }
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\student-portal\profile\profile.css
.profile-grid {
  display: grid;
  grid-template-columns: 160px 1fr;
  row-gap: 0.5rem;
  max-width: 480px;
}

.profile-grid dt {
  font-weight: 600;
  color: #4b5563;
}

.profile-grid dd {
  margin: 0;
}

.hint {
  color: #6b7280;
  font-size: 0.85rem;
}

.error {
  color: #dc2626;
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\student-portal\profile\profile.html
<h2>My Profile</h2>

@if (loading()) {
  <p>Loading...</p>
} @else if (error()) {
  <p class="error">{{ error() }}</p>
} @else {
  @if (student(); as s) {
    <dl class="profile-grid">
      <dt>Name</dt>
      <dd>{{ s.name }}</dd>
      <dt>Age</dt>
      <dd>{{ s.age }}</dd>
      <dt>Education</dt>
      <dd>{{ s.education }}</dd>
      <dt>Location</dt>
      <dd>{{ s.location ?? '—' }}</dd>
      <dt>Board</dt>
      <dd>{{ s.board ?? '—' }}</dd>
      <dt>Subjects</dt>
      <dd>{{ s.subjects?.join(', ') || '—' }}</dd>
    </dl>
    <p class="hint">Profile edits are managed by your counselor.</p>
  }
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\student-portal\profile\profile.ts
import { Component, inject, signal } from '@angular/core';

import { AuthService } from '../../../core/auth/auth.service';
import { Student } from '../../../models/student.model';
import { StudentService } from '../../../services/student.service';

@Component({
  selector: 'app-student-profile',
  standalone: true,
  templateUrl: './profile.html',
  styleUrl: './profile.css',
})
export class Profile {
  private readonly authService = inject(AuthService);
  private readonly studentService = inject(StudentService);

  protected readonly student = signal<Student | null>(null);
  protected readonly loading = signal(true);
  protected readonly error = signal<string | null>(null);

  constructor() {
    const studentId = this.authService.currentUser()?.student_id;
    if (!studentId) {
      this.error.set('No student profile is linked to your account.');
      this.loading.set(false);
      return;
    }

    this.studentService.getStudent(studentId).subscribe({
      next: (student) => {
        this.student.set(student);
        this.loading.set(false);
      },
      error: () => {
        this.error.set('Failed to load your profile.');
        this.loading.set(false);
      },
    });
  }
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\student-portal\progress\progress.css
.cards {
  display: flex;
  flex-direction: column;
  gap: 1rem;
  max-width: 480px;
}

.card {
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  padding: 1rem;
}

.bar {
  height: 10px;
  background: #e5e7eb;
  border-radius: 5px;
  overflow: hidden;
  margin-bottom: 0.5rem;
}

.bar-fill {
  height: 100%;
  background: #16a34a;
}

.summary {
  margin: 0;
  font-weight: 600;
}

.meta {
  margin: 0.25rem 0 0;
  font-size: 0.85rem;
  color: #6b7280;
}

.error {
  color: #dc2626;
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\student-portal\progress\progress.html
<h2>Progress Tracker</h2>

@if (loading()) {
  <p>Loading...</p>
} @else if (error()) {
  <p class="error">{{ error() }}</p>
} @else if (progress().length) {
  <div class="cards">
    @for (p of progress(); track p.tracking_id) {
      <div class="card">
        <div class="bar">
          <div class="bar-fill" [style.width.%]="p.completion_percentage"></div>
        </div>
        <p class="summary">
          {{ p.milestones_completed }} / {{ p.total_milestones }} milestones
          ({{ p.completion_percentage }}%)
        </p>
        @if (p.last_reassessed_date) {
          <p class="meta">Last reassessed {{ p.last_reassessed_date | date: 'mediumDate' }}</p>
        }
      </div>
    }
  </div>
} @else {
  <p>No progress recorded yet. Complete milestones on your roadmap to track progress here.</p>
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\student-portal\progress\progress.ts
import { DatePipe } from '@angular/common';
import { Component, inject, signal } from '@angular/core';

import { AuthService } from '../../../core/auth/auth.service';
import { Progress as ProgressModel } from '../../../models/progress.model';
import { ProgressService } from '../../../services/progress.service';

@Component({
  selector: 'app-progress',
  standalone: true,
  imports: [DatePipe],
  templateUrl: './progress.html',
  styleUrl: './progress.css',
})
export class Progress {
  private readonly authService = inject(AuthService);
  private readonly progressService = inject(ProgressService);

  protected readonly progress = signal<ProgressModel[]>([]);
  protected readonly loading = signal(true);
  protected readonly error = signal<string | null>(null);

  constructor() {
    const studentId = this.authService.currentUser()?.student_id;
    if (!studentId) {
      this.error.set('No student profile is linked to your account.');
      this.loading.set(false);
      return;
    }

    this.progressService.getForStudent(studentId).subscribe({
      next: (progress) => {
        this.progress.set(progress);
        this.loading.set(false);
      },
      error: () => {
        this.error.set('Failed to load your progress.');
        this.loading.set(false);
      },
    });
  }
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\student-portal\recommendations\recommendations.css
.cards {
  display: flex;
  flex-direction: column;
  gap: 1rem;
  max-width: 560px;
}

.card {
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  padding: 1rem;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.card-header h3 {
  margin: 0;
}

.score {
  font-weight: 600;
  color: #2563eb;
}

.rationale {
  color: #4b5563;
  font-size: 0.9rem;
}

.card button {
  margin-top: 0.5rem;
  padding: 0.4rem 0.8rem;
  background: #2563eb;
  color: #fff;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.error {
  color: #dc2626;
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\student-portal\recommendations\recommendations.html
<h2>Career Recommendations</h2>

@if (loading()) {
  <p>Loading...</p>
} @else if (error()) {
  <p class="error">{{ error() }}</p>
} @else if (recommendations().length) {
  <div class="cards">
    @for (rec of recommendations(); track rec.recommendation_id) {
      <div class="card">
        <div class="card-header">
          <h3>{{ rec.career_name }}</h3>
          <span class="score">{{ rec.match_score }}% match</span>
        </div>
        @if (rec.rationale) {
          <p class="rationale">{{ rec.rationale }}</p>
        }
        <button type="button" (click)="viewGaps(rec.recommendation_id)">View Skill Gaps</button>
      </div>
    }
  </div>
} @else {
  <p>No career recommendations yet. Check back after your assessment is complete.</p>
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\student-portal\recommendations\recommendations.ts
import { Component, inject, signal } from '@angular/core';
import { Router } from '@angular/router';

import { AuthService } from '../../../core/auth/auth.service';
import { CareerRecommendation } from '../../../models/recommendation.model';
import { RecommendationService } from '../../../services/recommendation.service';

@Component({
  selector: 'app-recommendations',
  standalone: true,
  templateUrl: './recommendations.html',
  styleUrl: './recommendations.css',
})
export class Recommendations {
  private readonly authService = inject(AuthService);
  private readonly recommendationService = inject(RecommendationService);
  private readonly router = inject(Router);

  protected readonly recommendations = signal<CareerRecommendation[]>([]);
  protected readonly loading = signal(true);
  protected readonly error = signal<string | null>(null);

  constructor() {
    const studentId = this.authService.currentUser()?.student_id;
    if (!studentId) {
      this.error.set('No student profile is linked to your account.');
      this.loading.set(false);
      return;
    }

    this.recommendationService.getForStudent(studentId).subscribe({
      next: (recommendations) => {
        this.recommendations.set(
          [...recommendations].sort((a, b) => b.match_score - a.match_score),
        );
        this.loading.set(false);
      },
      error: () => {
        this.error.set('Failed to load your career recommendations.');
        this.loading.set(false);
      },
    });
  }

  protected viewGaps(recommendationId: number): void {
    this.router.navigate(['/student/gap-analysis'], { queryParams: { recommendationId } });
  }
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\student-portal\roadmap\roadmap.css
.roadmap {
  max-width: 640px;
  margin-bottom: 2rem;
}

.meta {
  color: #6b7280;
  font-size: 0.85rem;
}

.timeline {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.milestone {
  border: 1px solid #e5e7eb;
  border-left: 4px solid #2563eb;
  border-radius: 6px;
  padding: 0.75rem 1rem;
}

.milestone.completed {
  border-left-color: #16a34a;
  background: #f0fdf4;
}

.milestone-header {
  display: flex;
  justify-content: space-between;
}

.status {
  font-size: 0.75rem;
  color: #6b7280;
}

.topics {
  margin: 0.4rem 0;
  padding-left: 1.2rem;
}

.project,
.resources,
.completed-date {
  font-size: 0.85rem;
  color: #4b5563;
  margin: 0.2rem 0;
}

.error {
  color: #dc2626;
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\student-portal\roadmap\roadmap.html
<h2>Learning Roadmap</h2>

@if (loading()) {
  <p>Loading...</p>
} @else if (error()) {
  <p class="error">{{ error() }}</p>
} @else if (roadmaps().length) {
  @for (roadmap of roadmaps(); track roadmap.roadmap_id) {
    <div class="roadmap">
      <h3>{{ roadmap.goal }}</h3>
      <p class="meta">{{ roadmap.duration_months }} months &middot; {{ roadmap.status }}</p>
      <div class="timeline">
        @for (milestone of roadmap.milestones; track milestone.milestone_id) {
          <div class="milestone" [class.completed]="milestone.status === 'COMPLETED'">
            <div class="milestone-header">
              <strong>{{ milestone.month_range }}</strong>
              <span class="status">{{ milestone.status }}</span>
            </div>
            <ul class="topics">
              @for (topic of milestone.topics; track topic) {
                <li>{{ topic }}</li>
              }
            </ul>
            @if (milestone.project) {
              <p class="project">Project: {{ milestone.project }}</p>
            }
            @if (milestone.resources?.length) {
              <p class="resources">Resources: {{ milestone.resources!.join(', ') }}</p>
            }
            @if (milestone.completed_date) {
              <p class="completed-date">Completed {{ milestone.completed_date | date: 'mediumDate' }}</p>
            }
          </div>
        }
      </div>
    </div>
  }
} @else {
  <p>No roadmap generated yet.</p>
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\student-portal\roadmap\roadmap.ts
import { DatePipe } from '@angular/common';
import { Component, inject, signal } from '@angular/core';

import { AuthService } from '../../../core/auth/auth.service';
import { Roadmap as RoadmapModel } from '../../../models/roadmap.model';
import { RoadmapService } from '../../../services/roadmap.service';

@Component({
  selector: 'app-roadmap',
  standalone: true,
  imports: [DatePipe],
  templateUrl: './roadmap.html',
  styleUrl: './roadmap.css',
})
export class Roadmap {
  private readonly authService = inject(AuthService);
  private readonly roadmapService = inject(RoadmapService);

  protected readonly roadmaps = signal<RoadmapModel[]>([]);
  protected readonly loading = signal(true);
  protected readonly error = signal<string | null>(null);

  constructor() {
    const studentId = this.authService.currentUser()?.student_id;
    if (!studentId) {
      this.error.set('No student profile is linked to your account.');
      this.loading.set(false);
      return;
    }

    this.roadmapService.getForStudent(studentId).subscribe({
      next: (roadmaps) => {
        this.roadmaps.set(roadmaps);
        this.loading.set(false);
      },
      error: () => {
        this.error.set('Failed to load your roadmap.');
        this.loading.set(false);
      },
    });
  }
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\student-portal\skills\skills.css
.skill-list {
  list-style: none;
  padding: 0;
  max-width: 480px;
}

.skill-list li {
  display: grid;
  grid-template-columns: 140px 80px 1fr 48px;
  align-items: center;
  gap: 0.5rem;
  margin-bottom: 0.5rem;
}

.skill-type {
  font-size: 0.75rem;
  color: #6b7280;
}

.bar {
  height: 8px;
  background: #e5e7eb;
  border-radius: 4px;
  overflow: hidden;
}

.bar-fill {
  height: 100%;
  background: #2563eb;
}

.skill-score {
  font-size: 0.85rem;
  text-align: right;
}

.metrics-grid {
  display: grid;
  grid-template-columns: 200px 1fr;
  row-gap: 0.5rem;
  max-width: 480px;
}

.metrics-grid dt {
  font-weight: 600;
  color: #4b5563;
}

.metrics-grid dd {
  margin: 0;
}

.error {
  color: #dc2626;
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\student-portal\skills\skills.html
<h2>Skill Assessment</h2>

@if (loading()) {
  <p>Loading...</p>
} @else if (error()) {
  <p class="error">{{ error() }}</p>
}

@if (!loading()) {
  <h3>Skills</h3>
  @if (skills().length) {
    <ul class="skill-list">
      @for (skill of skills(); track skill.id) {
        <li>
          <span class="skill-name">{{ skill.skill_name }}</span>
          <span class="skill-type">{{ skill.skill_type }}</span>
          <div class="bar">
            <div class="bar-fill" [style.width.%]="skill.proficiency_score"></div>
          </div>
          <span class="skill-score">{{ skill.proficiency_score }}%</span>
        </li>
      }
    </ul>
  } @else {
    <p>No skills assessed yet.</p>
  }

  <h3>Behavioral Metrics</h3>
  @if (latestMetric(); as m) {
    <dl class="metrics-grid">
      <dt>Learning Speed</dt>
      <dd>{{ m.learning_speed ?? '—' }}</dd>
      <dt>Consistency</dt>
      <dd>{{ m.consistency ?? '—' }}</dd>
      <dt>Curiosity Index</dt>
      <dd>{{ m.curiosity_index ?? '—' }}</dd>
      <dt>Persistence</dt>
      <dd>{{ m.persistence ?? '—' }}</dd>
      <dt>Goal Completion Rate</dt>
      <dd>{{ m.goal_completion_rate ?? '—' }}</dd>
    </dl>
  } @else {
    <p>No behavioral metrics recorded yet.</p>
  }
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\student-portal\skills\skills.ts
import { Component, inject, signal } from '@angular/core';

import { AuthService } from '../../../core/auth/auth.service';
import { BehavioralMetric, StudentSkill } from '../../../models/skill.model';
import { SkillService } from '../../../services/skill.service';

@Component({
  selector: 'app-student-skills',
  standalone: true,
  templateUrl: './skills.html',
  styleUrl: './skills.css',
})
export class Skills {
  private readonly authService = inject(AuthService);
  private readonly skillService = inject(SkillService);

  protected readonly skills = signal<StudentSkill[]>([]);
  protected readonly behavioralMetrics = signal<BehavioralMetric[]>([]);
  protected readonly loading = signal(true);
  protected readonly error = signal<string | null>(null);

  constructor() {
    const studentId = this.authService.currentUser()?.student_id;
    if (!studentId) {
      this.error.set('No student profile is linked to your account.');
      this.loading.set(false);
      return;
    }

    this.skillService.getSkills(studentId).subscribe({
      next: (skills) => this.skills.set(skills),
      error: () => this.error.set('Failed to load your skills.'),
    });

    this.skillService.getBehavioralMetrics(studentId).subscribe({
      next: (metrics) => {
        this.behavioralMetrics.set(metrics);
        this.loading.set(false);
      },
      error: () => {
        this.error.set('Failed to load your behavioral metrics.');
        this.loading.set(false);
      },
    });
  }

  protected latestMetric(): BehavioralMetric | null {
    const metrics = this.behavioralMetrics();
    return metrics.length ? metrics[metrics.length - 1] : null;
  }
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\student-portal\student-shell\student-shell.css
.portal-layout {
  display: flex;
  min-height: calc(100vh - 56px);
}

.portal-nav {
  width: 220px;
  flex-shrink: 0;
  display: flex;
  flex-direction: column;
  padding: 1rem 0;
  background: #f9fafb;
  border-right: 1px solid #e5e7eb;
}

.portal-nav a {
  padding: 0.6rem 1.25rem;
  color: #374151;
  text-decoration: none;
  font-size: 0.9rem;
}

.portal-nav a:hover {
  background: #f3f4f6;
}

.portal-nav a.active {
  background: #e0e7ff;
  color: #2563eb;
  font-weight: 600;
}

.portal-content {
  flex: 1;
  padding: 1.5rem;
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\student-portal\student-shell\student-shell.html
<app-header portalTitle="Student Portal" />
<div class="portal-layout">
  <nav class="portal-nav">
    @for (link of navLinks; track link.path) {
      <a [routerLink]="link.path" routerLinkActive="active" [routerLinkActiveOptions]="{ exact: link.path === '' }">
        {{ link.label }}
      </a>
    }
  </nav>
  <main class="portal-content">
    <router-outlet />
  </main>
</div>

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\student-portal\student-shell\student-shell.ts
import { Component } from '@angular/core';
import { RouterLink, RouterLinkActive, RouterOutlet } from '@angular/router';

import { Header } from '../../../shared/layout/header/header';

@Component({
  selector: 'app-student-shell',
  standalone: true,
  imports: [RouterOutlet, RouterLink, RouterLinkActive, Header],
  templateUrl: './student-shell.html',
  styleUrl: './student-shell.css',
})
export class StudentShell {
  protected readonly navLinks = [
    { path: '', label: 'Dashboard' },
    { path: 'profile', label: 'Profile' },
    { path: 'academic-history', label: 'Academic & Learning History' },
    { path: 'skills', label: 'Skill Assessment' },
    { path: 'discovery', label: 'Discovery Questions' },
    { path: 'recommendations', label: 'Career Recommendations' },
    { path: 'gap-analysis', label: 'Skill Gap Analysis' },
    { path: 'roadmap', label: 'Roadmap' },
    { path: 'mentor-chat', label: 'Mentor Chat' },
    { path: 'progress', label: 'Progress Tracker' },
  ];
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\unauthorized\unauthorized.ts
import { Component } from '@angular/core';
import { RouterLink } from '@angular/router';

@Component({
  selector: 'app-unauthorized',
  standalone: true,
  imports: [RouterLink],
  template: `
    <div class="unauthorized">
      <h1>403 - Not authorized</h1>
      <p>You don't have permission to view this page.</p>
      <a routerLink="/login">Back to login</a>
    </div>
  `,
  styles: [
    `
      .unauthorized {
        display: flex;
        flex-direction: column;
        align-items: center;
        justify-content: center;
        min-height: 100vh;
        gap: 0.5rem;
      }
    `,
  ],
})
export class Unauthorized {}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\features\.gitkeep

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\models\.gitkeep
C:\ws\agent\admission\counselor-agent\frontend\src\app\models\analytics.model.ts
export interface CareerDemandPoint {
  career_name: string;
  market_demand_score: number;
}

export interface CareerDemandTrends {
  chart_type: string;
  data: CareerDemandPoint[];
}

export interface SkillGapHeatmapEntry {
  career_name: string;
  skill_gap_counts: Record<string, number>;
}

export interface SkillGapHeatmap {
  chart_type: string;
  data: SkillGapHeatmapEntry[];
}

export interface CareerCompletionRate {
  career_name: string;
  average_completion_percentage: number;
}

export interface RoadmapCompletionRate {
  chart_type: string;
  overall_average_completion_percentage: number;
  by_career: CareerCompletionRate[];
}

export interface TopRecommendedCareer {
  career_name: string;
  recommendation_count: number;
}

export interface TopRecommendedCareers {
  chart_type: string;
  data: TopRecommendedCareer[];
}

export interface FunnelStage {
  stage: string;
  count: number;
}

export interface CertificationUptake {
  chart_type: string;
  data: FunnelStage[];
}

export interface DashboardSummary {
  career_demand_trends: CareerDemandTrends;
  cohort_skill_gap_heatmap: SkillGapHeatmap;
  roadmap_completion_rate: RoadmapCompletionRate;
  top_recommended_careers: TopRecommendedCareers;
  certification_uptake: CertificationUptake;
  active_students: number;
  assessments_completed_today: number;
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\models\auth.model.ts
export interface LoginRequest {
  email: string;
  password: string;
}

export interface TokenResponse {
  access_token: string;
  refresh_token: string;
  token_type: string;
}

export interface AccessTokenResponse {
  access_token: string;
  token_type: string;
}

export interface CurrentUser {
  user_id: number;
  email: string;
  full_name: string;
  role: string;
  is_active: boolean;
  student_id: number | null;
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\models\catalog.model.ts
export interface CareerDefinition {
  career_name: string;
  required_skills: string[];
  target_skills: string[];
  interest_map: Record<string, string>;
  academic_subjects: string[];
  market_demand_score: number;
}

export interface Certification {
  certification_id: number;
  career_name: string;
  certification_name: string;
  provider: string | null;
}

export interface CertificationCreate {
  career_name: string;
  certification_name: string;
  provider?: string | null;
}

export interface CertificationUpdate {
  career_name?: string;
  certification_name?: string;
  provider?: string | null;
}

export interface Project {
  project_id: number;
  career_name: string;
  project_name: string;
  description: string | null;
  difficulty_level: string | null;
}

export interface ProjectCreate {
  career_name: string;
  project_name: string;
  description?: string | null;
  difficulty_level?: string | null;
}

export interface ProjectUpdate {
  career_name?: string;
  project_name?: string;
  description?: string | null;
  difficulty_level?: string | null;
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\models\discovery.model.ts
export interface DiscoveryQuestion {
  question_id: number;
  category: string;
  question_text: string;
  options: Record<string, string>;
  version: number;
}

export interface DiscoveryAnswerRequest {
  student_id: number;
  question_id: number;
  selected_option: string;
}

export interface DiscoveryResponse {
  response_id: number;
  student_id: number;
  question_id: number;
  selected_option: string;
}

export interface DiscoveryQuestionCreate {
  category: string;
  question_text: string;
  options: Record<string, string>;
}

export interface DiscoveryQuestionUpdate {
  category?: string;
  question_text?: string;
  options?: Record<string, string>;
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\models\gap-analysis.model.ts
export interface SkillGap {
  gap_id: number;
  skill_id: number;
  skill_name: string;
  status: string;
}

export interface GapAnalysisResponse {
  recommendation_id: number;
  matched: string[];
  missing: string[];
  details: SkillGap[];
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\models\history.model.ts
export interface AcademicHistory {
  history_id: number;
  student_id: number;
  subject: string;
  score: number;
  attendance: number | null;
  learning_behavior: string | null;
  recorded_date: string;
}

export interface LearningPlatformData {
  record_id: number;
  student_id: number;
  platform: string;
  courses_completed: number;
  hours_spent: number;
  skill_area: string | null;
  recorded_date: string;
}

export interface StudentHistory {
  academic_history: AcademicHistory[];
  learning_platform_data: LearningPlatformData[];
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\models\mentor.model.ts
export interface MentorChatRequest {
  studentId: number;
  message: string;
}

export interface MentorChatResponse {
  reply: string;
  relatedCareer: string | null;
}

export interface MentorConversation {
  conversation_id: number;
  student_id: number;
  message: string;
  reply: string | null;
  related_career: string | null;
  created_date: string;
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\models\progress.model.ts
export interface UpdatedRecommendation {
  career_name: string;
  match_score: number;
}

export interface ProgressCompleteResponse {
  tracking_id: number;
  student_id: number;
  roadmap_id: number;
  milestones_completed: number;
  total_milestones: number;
  completion_percentage: number;
  last_reassessed_date: string | null;
  reassessment_triggered: boolean;
  updated_recommendation: UpdatedRecommendation | null;
}

export interface Progress {
  tracking_id: number;
  student_id: number;
  roadmap_id: number;
  milestones_completed: number;
  total_milestones: number;
  completion_percentage: number;
  last_reassessed_date: string | null;
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\models\recommendation.model.ts
export interface CareerRecommendation {
  recommendation_id: number;
  student_id: number;
  career_name: string;
  match_score: number;
  rationale: string | null;
  created_date: string;
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\models\roadmap.model.ts
export interface RoadmapMilestone {
  milestone_id: number;
  roadmap_id: number;
  month_range: string;
  topics: string[];
  resources: string[] | null;
  project: string | null;
  status: string;
  completed_date: string | null;
}

export interface Roadmap {
  roadmap_id: number;
  student_id: number;
  recommendation_id: number;
  goal: string;
  duration_months: number;
  status: string;
  created_date: string;
  milestones: RoadmapMilestone[];
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\models\skill.model.ts
export interface StudentSkill {
  id: number;
  student_id: number;
  skill_id: number;
  skill_name: string;
  skill_type: string;
  proficiency_score: number;
  assessed_date: string;
}

export interface BehavioralMetric {
  metric_id: number;
  student_id: number;
  learning_speed: number | null;
  consistency: number | null;
  curiosity_index: number | null;
  persistence: number | null;
  goal_completion_rate: number | null;
  recorded_date: string;
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\models\student.model.ts
export interface Student {
  student_id: number;
  name: string;
  age: number;
  education: string;
  location: string | null;
  board: string | null;
  subjects: string[] | null;
  created_date: string;
}

export interface StudentCreate {
  name: string;
  age: number;
  education: string;
  location?: string | null;
  board?: string | null;
  subjects?: string[] | null;
}

export interface StudentUpdate {
  name?: string;
  age?: number;
  education?: string;
  location?: string | null;
  board?: string | null;
  subjects?: string[] | null;
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\services\.gitkeep
C:\ws\agent\admission\counselor-agent\frontend\src\app\services\analytics.service.ts
import { HttpClient } from '@angular/common/http';
import { Injectable, inject } from '@angular/core';
import { Observable } from 'rxjs';

import { environment } from '../../environments/environment';
import {
  CareerDemandTrends,
  CertificationUptake,
  DashboardSummary,
  RoadmapCompletionRate,
  SkillGapHeatmap,
  TopRecommendedCareers,
} from '../models/analytics.model';

@Injectable({ providedIn: 'root' })
export class AnalyticsService {
  private readonly http = inject(HttpClient);
  private readonly baseUrl = `${environment.apiUrl}/analytics`;

  getDashboard(): Observable<DashboardSummary> {
    return this.http.get<DashboardSummary>(`${this.baseUrl}/dashboard`);
  }

  getCareerDemandTrends(): Observable<CareerDemandTrends> {
    return this.http.get<CareerDemandTrends>(`${this.baseUrl}/career-demand-trends`);
  }

  getSkillGapHeatmap(): Observable<SkillGapHeatmap> {
    return this.http.get<SkillGapHeatmap>(`${this.baseUrl}/skill-gap-heatmap`);
  }

  getRoadmapCompletionRate(): Observable<RoadmapCompletionRate> {
    return this.http.get<RoadmapCompletionRate>(`${this.baseUrl}/roadmap-completion-rate`);
  }

  getTopRecommendedCareers(): Observable<TopRecommendedCareers> {
    return this.http.get<TopRecommendedCareers>(`${this.baseUrl}/top-recommended-careers`);
  }

  getCertificationUptake(): Observable<CertificationUptake> {
    return this.http.get<CertificationUptake>(`${this.baseUrl}/certification-uptake`);
  }
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\services\catalog.service.ts
import { HttpClient } from '@angular/common/http';
import { Injectable, inject } from '@angular/core';
import { Observable } from 'rxjs';

import { environment } from '../../environments/environment';
import {
  CareerDefinition,
  Certification,
  CertificationCreate,
  CertificationUpdate,
  Project,
  ProjectCreate,
  ProjectUpdate,
} from '../models/catalog.model';

@Injectable({ providedIn: 'root' })
export class CatalogService {
  private readonly http = inject(HttpClient);
  private readonly baseUrl = `${environment.apiUrl}/catalog`;

  getCareers(): Observable<CareerDefinition[]> {
    return this.http.get<CareerDefinition[]>(`${this.baseUrl}/careers`);
  }

  getCertifications(): Observable<Certification[]> {
    return this.http.get<Certification[]>(`${this.baseUrl}/certifications`);
  }

  createCertification(payload: CertificationCreate): Observable<Certification> {
    return this.http.post<Certification>(`${this.baseUrl}/certifications`, payload);
  }

  updateCertification(certificationId: number, payload: CertificationUpdate): Observable<Certification> {
    return this.http.put<Certification>(`${this.baseUrl}/certifications/${certificationId}`, payload);
  }

  deleteCertification(certificationId: number): Observable<void> {
    return this.http.delete<void>(`${this.baseUrl}/certifications/${certificationId}`);
  }

  getProjects(): Observable<Project[]> {
    return this.http.get<Project[]>(`${this.baseUrl}/projects`);
  }

  createProject(payload: ProjectCreate): Observable<Project> {
    return this.http.post<Project>(`${this.baseUrl}/projects`, payload);
  }

  updateProject(projectId: number, payload: ProjectUpdate): Observable<Project> {
    return this.http.put<Project>(`${this.baseUrl}/projects/${projectId}`, payload);
  }

  deleteProject(projectId: number): Observable<void> {
    return this.http.delete<void>(`${this.baseUrl}/projects/${projectId}`);
  }
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\services\discovery.service.ts
import { HttpClient } from '@angular/common/http';
import { Injectable, inject } from '@angular/core';
import { Observable } from 'rxjs';

import { environment } from '../../environments/environment';
import {
  DiscoveryAnswerRequest,
  DiscoveryQuestion,
  DiscoveryQuestionCreate,
  DiscoveryQuestionUpdate,
  DiscoveryResponse,
} from '../models/discovery.model';

@Injectable({ providedIn: 'root' })
export class DiscoveryService {
  private readonly http = inject(HttpClient);
  private readonly baseUrl = `${environment.apiUrl}/discovery`;

  getNextQuestion(studentId: number): Observable<DiscoveryQuestion | null> {
    return this.http.get<DiscoveryQuestion | null>(`${this.baseUrl}/next-question`, {
      params: { studentId },
    });
  }

  submitAnswer(request: DiscoveryAnswerRequest): Observable<DiscoveryResponse> {
    return this.http.post<DiscoveryResponse>(`${this.baseUrl}/answer`, request);
  }

  listQuestions(): Observable<DiscoveryQuestion[]> {
    return this.http.get<DiscoveryQuestion[]>(`${this.baseUrl}/questions`);
  }

  createQuestion(payload: DiscoveryQuestionCreate): Observable<DiscoveryQuestion> {
    return this.http.post<DiscoveryQuestion>(`${this.baseUrl}/questions`, payload);
  }

  updateQuestion(questionId: number, payload: DiscoveryQuestionUpdate): Observable<DiscoveryQuestion> {
    return this.http.put<DiscoveryQuestion>(`${this.baseUrl}/questions/${questionId}`, payload);
  }
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\services\gap-analysis.service.ts
import { HttpClient } from '@angular/common/http';
import { Injectable, inject } from '@angular/core';
import { Observable } from 'rxjs';

import { environment } from '../../environments/environment';
import { GapAnalysisResponse } from '../models/gap-analysis.model';

@Injectable({ providedIn: 'root' })
export class GapAnalysisService {
  private readonly http = inject(HttpClient);
  private readonly baseUrl = `${environment.apiUrl}/gap-analysis`;

  getByRecommendation(recommendationId: number): Observable<GapAnalysisResponse> {
    return this.http.get<GapAnalysisResponse>(`${this.baseUrl}/${recommendationId}`);
  }
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\services\mentor.service.ts
import { HttpClient } from '@angular/common/http';
import { Injectable, inject } from '@angular/core';
import { Observable } from 'rxjs';

import { environment } from '../../environments/environment';
import { MentorChatRequest, MentorChatResponse, MentorConversation } from '../models/mentor.model';

@Injectable({ providedIn: 'root' })
export class MentorService {
  private readonly http = inject(HttpClient);
  private readonly baseUrl = `${environment.apiUrl}/mentor`;

  chat(request: MentorChatRequest): Observable<MentorChatResponse> {
    return this.http.post<MentorChatResponse>(`${this.baseUrl}/chat`, request);
  }

  getConversations(studentId: number): Observable<MentorConversation[]> {
    return this.http.get<MentorConversation[]>(`${this.baseUrl}/students/${studentId}`);
  }
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\services\progress.service.ts
import { HttpClient } from '@angular/common/http';
import { Injectable, inject } from '@angular/core';
import { Observable } from 'rxjs';

import { environment } from '../../environments/environment';
import { Progress } from '../models/progress.model';

@Injectable({ providedIn: 'root' })
export class ProgressService {
  private readonly http = inject(HttpClient);
  private readonly baseUrl = `${environment.apiUrl}/progress`;

  getForStudent(studentId: number): Observable<Progress[]> {
    return this.http.get<Progress[]>(`${this.baseUrl}/${studentId}`);
  }
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\services\recommendation.service.ts
import { HttpClient } from '@angular/common/http';
import { Injectable, inject } from '@angular/core';
import { Observable } from 'rxjs';

import { environment } from '../../environments/environment';
import { CareerRecommendation } from '../models/recommendation.model';

@Injectable({ providedIn: 'root' })
export class RecommendationService {
  private readonly http = inject(HttpClient);
  private readonly baseUrl = `${environment.apiUrl}/recommendations`;

  getForStudent(studentId: number): Observable<CareerRecommendation[]> {
    return this.http.get<CareerRecommendation[]>(`${this.baseUrl}/students/${studentId}`);
  }
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\services\roadmap.service.ts
import { HttpClient } from '@angular/common/http';
import { Injectable, inject } from '@angular/core';
import { Observable } from 'rxjs';

import { environment } from '../../environments/environment';
import { Roadmap } from '../models/roadmap.model';

@Injectable({ providedIn: 'root' })
export class RoadmapService {
  private readonly http = inject(HttpClient);
  private readonly baseUrl = `${environment.apiUrl}/roadmaps`;

  getForStudent(studentId: number): Observable<Roadmap[]> {
    return this.http.get<Roadmap[]>(`${this.baseUrl}/students/${studentId}`);
  }
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\services\skill.service.ts
import { HttpClient } from '@angular/common/http';
import { Injectable, inject } from '@angular/core';
import { Observable } from 'rxjs';

import { environment } from '../../environments/environment';
import { BehavioralMetric, StudentSkill } from '../models/skill.model';

@Injectable({ providedIn: 'root' })
export class SkillService {
  private readonly http = inject(HttpClient);
  private readonly baseUrl = `${environment.apiUrl}/students`;

  getSkills(studentId: number): Observable<StudentSkill[]> {
    return this.http.get<StudentSkill[]>(`${this.baseUrl}/${studentId}/skills`);
  }

  getBehavioralMetrics(studentId: number): Observable<BehavioralMetric[]> {
    return this.http.get<BehavioralMetric[]>(`${this.baseUrl}/${studentId}/behavioral-metrics`);
  }
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\services\student.service.ts
import { HttpClient } from '@angular/common/http';
import { Injectable, inject } from '@angular/core';
import { Observable } from 'rxjs';

import { environment } from '../../environments/environment';
import { StudentHistory } from '../models/history.model';
import { Student, StudentCreate, StudentUpdate } from '../models/student.model';

@Injectable({ providedIn: 'root' })
export class StudentService {
  private readonly http = inject(HttpClient);
  private readonly baseUrl = `${environment.apiUrl}/students`;

  listStudents(skip = 0, limit = 50): Observable<Student[]> {
    return this.http.get<Student[]>(this.baseUrl, { params: { skip, limit } });
  }

  createStudent(payload: StudentCreate): Observable<Student> {
    return this.http.post<Student>(this.baseUrl, payload);
  }

  deleteStudent(studentId: number): Observable<void> {
    return this.http.delete<void>(`${this.baseUrl}/${studentId}`);
  }

  getStudent(studentId: number): Observable<Student> {
    return this.http.get<Student>(`${this.baseUrl}/${studentId}`);
  }

  updateStudent(studentId: number, update: StudentUpdate): Observable<Student> {
    return this.http.put<Student>(`${this.baseUrl}/${studentId}`, update);
  }

  getHistory(studentId: number): Observable<StudentHistory> {
    return this.http.get<StudentHistory>(`${this.baseUrl}/${studentId}/history`);
  }
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\shared\layout\header\header.css
.app-header {
  display: flex;
  align-items: center;
  gap: 1rem;
  padding: 0.75rem 1.5rem;
  background: #1f2937;
  color: #fff;
}

.portal-title {
  font-weight: 600;
  flex: 1;
}

.user-info {
  font-size: 0.9rem;
  opacity: 0.85;
}

.app-header button {
  background: #374151;
  color: #fff;
  border: none;
  border-radius: 4px;
  padding: 0.4rem 0.8rem;
  cursor: pointer;
}

.app-header button:hover {
  background: #4b5563;
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\shared\layout\header\header.html
<header class="app-header">
  <span class="portal-title">{{ portalTitle() }}</span>
  @if (currentUser(); as user) {
    <span class="user-info">{{ user.full_name }} ({{ user.role }})</span>
    <button type="button" (click)="logout()">Log out</button>
  }
</header>

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\shared\layout\header\header.ts
import { Component, inject, input } from '@angular/core';

import { AuthService } from '../../../core/auth/auth.service';

@Component({
  selector: 'app-header',
  standalone: true,
  templateUrl: './header.html',
  styleUrl: './header.css',
})
export class Header {
  readonly portalTitle = input('Career Counselor Agent');

  private readonly authService = inject(AuthService);
  readonly currentUser = this.authService.currentUser;

  protected logout(): void {
    this.authService.logout();
  }
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\shared\.gitkeep

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\app.config.ts
import { HttpClient, provideHttpClient, withInterceptors } from '@angular/common/http';
import { ApplicationConfig, inject, provideAppInitializer, provideZoneChangeDetection } from '@angular/core';
import { provideRouter } from '@angular/router';

import { routes } from './app.routes';
import { authInterceptor } from './core/auth/auth.interceptor';
import { AuthService } from './core/auth/auth.service';
import { mockApiInterceptor } from './mocks/mock-api.interceptor';

export const appConfig: ApplicationConfig = {
  providers: [
    provideZoneChangeDetection({ eventCoalescing: true }),
    provideRouter(routes),
    // authInterceptor must run first so the Authorization header is attached before
    // mockApiInterceptor inspects it (e.g. for /auth/me).
    provideHttpClient(withInterceptors([authInterceptor, mockApiInterceptor])),
    provideAppInitializer(() => inject(AuthService).restoreSession()),
  ],
};

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\app.css
C:\ws\agent\admission\counselor-agent\frontend\src\app\app.html
<router-outlet />

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\app.routes.ts
import { Routes } from '@angular/router';

import { authGuard } from './core/auth/auth.guard';
import { roleGuard } from './core/auth/role.guard';
import { ROLES } from './core/roles';

export const routes: Routes = [
  { path: '', pathMatch: 'full', redirectTo: 'login' },
  {
    path: 'login',
    loadComponent: () => import('./features/auth/login/login').then((m) => m.Login),
  },
  {
    path: 'unauthorized',
    loadComponent: () => import('./features/unauthorized/unauthorized').then((m) => m.Unauthorized),
  },
  {
    path: 'student',
    canActivate: [authGuard, roleGuard(ROLES.STUDENT)],
    loadComponent: () =>
      import('./features/student-portal/student-shell/student-shell').then((m) => m.StudentShell),
    children: [
      {
        path: '',
        loadComponent: () => import('./features/student-portal/dashboard/dashboard').then((m) => m.Dashboard),
      },
      {
        path: 'profile',
        loadComponent: () => import('./features/student-portal/profile/profile').then((m) => m.Profile),
      },
      {
        path: 'academic-history',
        loadComponent: () =>
          import('./features/student-portal/academic-history/academic-history').then((m) => m.AcademicHistory),
      },
      {
        path: 'skills',
        loadComponent: () => import('./features/student-portal/skills/skills').then((m) => m.Skills),
      },
      {
        path: 'discovery',
        loadComponent: () => import('./features/student-portal/discovery/discovery').then((m) => m.Discovery),
      },
      {
        path: 'recommendations',
        loadComponent: () =>
          import('./features/student-portal/recommendations/recommendations').then((m) => m.Recommendations),
      },
      {
        path: 'gap-analysis',
        loadComponent: () =>
          import('./features/student-portal/gap-analysis/gap-analysis').then((m) => m.GapAnalysis),
      },
      {
        path: 'roadmap',
        loadComponent: () => import('./features/student-portal/roadmap/roadmap').then((m) => m.Roadmap),
      },
      {
        path: 'mentor-chat',
        loadComponent: () =>
          import('./features/student-portal/mentor-chat/mentor-chat').then((m) => m.MentorChat),
      },
      {
        path: 'progress',
        loadComponent: () => import('./features/student-portal/progress/progress').then((m) => m.Progress),
      },
    ],
  },
  {
    path: 'counselor',
    canActivate: [authGuard, roleGuard(ROLES.ADMIN, ROLES.COUNSELOR, ROLES.MENTOR, ROLES.READ_ONLY)],
    loadComponent: () =>
      import('./features/counselor-portal/counselor-shell/counselor-shell').then((m) => m.CounselorShell),
    children: [
      {
        path: '',
        loadComponent: () =>
          import('./features/counselor-portal/dashboard/dashboard').then((m) => m.Dashboard),
      },
      {
        path: 'students',
        loadComponent: () =>
          import('./features/counselor-portal/student-management/student-management').then(
            (m) => m.StudentManagement,
          ),
      },
      {
        path: 'students/:id',
        loadComponent: () =>
          import('./features/counselor-portal/student-management/student-detail/student-detail').then(
            (m) => m.StudentDetail,
          ),
      },
      {
        path: 'question-bank',
        loadComponent: () =>
          import('./features/counselor-portal/question-bank/question-bank').then((m) => m.QuestionBank),
      },
      {
        path: 'career-criteria',
        loadComponent: () =>
          import('./features/counselor-portal/career-criteria/career-criteria').then((m) => m.CareerCriteria),
      },
      {
        path: 'analytics',
        loadComponent: () =>
          import('./features/counselor-portal/analytics/analytics').then((m) => m.Analytics),
      },
    ],
  },
  { path: '**', redirectTo: 'login' },
];

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\app\app.ts
import { Component } from '@angular/core';
import { RouterOutlet } from '@angular/router';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [RouterOutlet],
  templateUrl: './app.html',
  styleUrl: './app.css',
})
export class App {}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\assets\.gitkeep

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\environments\environment.development.ts
export const environment = {
  production: false,
  apiUrl: 'http://localhost:8000/api/v1',
  // Toggle to serve fixture data from the mock interceptor instead of calling the real backend.
  useMockData: true,
};

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\environments\environment.ts
export const environment = {
  production: true,
  apiUrl: '/api/v1',
  // Toggle to serve fixture data from the mock interceptor instead of calling the real backend.
  useMockData: false,
};

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\index.html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>Career Counselor Agent</title>
  <base href="/">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <link rel="icon" type="image/x-icon" href="favicon.ico">
</head>
<body>
  <app-root></app-root>
</body>
</html>

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { appConfig } from './app/app.config';
import { App } from './app/app';

bootstrapApplication(App, appConfig).catch((err) => console.error(err));

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\src\styles.css
html, body {
  height: 100%;
  margin: 0;
  font-family: Roboto, "Helvetica Neue", sans-serif;
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\angular.json
{
  "$schema": "./node_modules/@angular/cli/lib/config/schema.json",
  "version": 1,
  "newProjectRoot": "projects",
  "projects": {
    "frontend": {
      "projectType": "application",
      "schematics": {},
      "root": "",
      "sourceRoot": "src",
      "prefix": "app",
      "architect": {
        "build": {
          "builder": "@angular-devkit/build-angular:application",
          "options": {
            "outputPath": "dist/frontend",
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
                { "type": "initial", "maximumWarning": "500kb", "maximumError": "1mb" },
                { "type": "anyComponentStyle", "maximumWarning": "4kb", "maximumError": "8kb" }
              ],
              "outputHashing": "all"
            },
            "development": {
              "optimization": false,
              "extractLicenses": false,
              "sourceMap": true,
              "namedChunks": true,
              "fileReplacements": [
                {
                  "replace": "src/environments/environment.ts",
                  "with": "src/environments/environment.development.ts"
                }
              ]
            }
          },
          "defaultConfiguration": "production"
        },
        "serve": {
          "builder": "@angular-devkit/build-angular:dev-server",
          "configurations": {
            "production": { "buildTarget": "frontend:build:production" },
            "development": { "buildTarget": "frontend:build:development" }
          },
          "defaultConfiguration": "development"
        },
        "test": {
          "builder": "@angular-devkit/build-angular:karma",
          "options": {
            "polyfills": ["zone.js", "zone.js/testing"],
            "tsConfig": "tsconfig.spec.json",
            "assets": ["src/assets"],
            "styles": ["src/styles.css"],
            "scripts": []
          }
        }
      }
    }
  }
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\Dockerfile
FROM node:22-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build -- --configuration production

FROM nginx:alpine
COPY --from=build /app/dist/frontend/browser /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\package.json
{
  "name": "counselor-agent-frontend",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "ng": "ng",
    "start": "ng serve",
    "build": "ng build",
    "watch": "ng build --watch --configuration development",
    "test": "ng test"
  },
  "dependencies": {
    "@angular/animations": "^19.0.0",
    "@angular/common": "^19.0.0",
    "@angular/compiler": "^19.0.0",
    "@angular/core": "^19.0.0",
    "@angular/forms": "^19.0.0",
    "@angular/platform-browser": "^19.0.0",
    "@angular/platform-browser-dynamic": "^19.0.0",
    "@angular/router": "^19.0.0",
    "rxjs": "~7.8.0",
    "tslib": "^2.6.0",
    "zone.js": "~0.15.0"
  },
  "devDependencies": {
    "@angular-devkit/build-angular": "^19.0.0",
    "@angular/cli": "^19.0.0",
    "@angular/compiler-cli": "^19.0.0",
    "typescript": "~5.6.0"
  }
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\tsconfig.app.json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "outDir": "./out-tsc/app",
    "types": []
  },
  "files": ["src/main.ts"],
  "include": ["src/**/*.d.ts"]
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\tsconfig.json
{
  "compileOnSave": false,
  "compilerOptions": {
    "outDir": "./dist/out-tsc",
    "strict": true,
    "noImplicitOverride": true,
    "noPropertyAccessFromIndexSignature": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "skipLibCheck": true,
    "isolatedModules": true,
    "esModuleInterop": true,
    "resolveJsonModule": true,
    "experimentalDecorators": true,
    "moduleResolution": "bundler",
    "importHelpers": true,
    "target": "ES2022",
    "module": "ES2022"
  },
  "angularCompilerOptions": {
    "enableI18nLegacyMessageIdFormat": false,
    "strictInjectionParameters": true,
    "strictInputAccessModifiers": true,
    "strictTemplates": true
  }
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\frontend\tsconfig.spec.json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "outDir": "./out-tsc/spec",
    "types": ["jasmine"]
  },
  "include": ["src/**/*.spec.ts", "src/**/*.d.ts"]
}

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\infrastructure\docker\.gitkeep

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\infrastructure\gcp\.gitkeep

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\infrastructure\terraform\.gitkeep

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\scripts\create_admin.py
"""Seeds the roles catalog (if missing) and creates an ADMIN user.

Usage:
    python scripts/create_admin.py <email> <password> [full_name]
"""
import sys
from pathlib import Path

BACKEND_DIR = Path(__file__).resolve().parent.parent / "backend"
sys.path.insert(0, str(BACKEND_DIR))

from database.base import Base  # noqa: E402
from database.session import SessionLocal, engine  # noqa: E402
from models import Role, User  # noqa: E402
from security.password import hash_password  # noqa: E402
from security.roles import ADMIN, ALL_ROLES  # noqa: E402


def ensure_roles(db) -> None:
    for role_name in ALL_ROLES:
        if not db.query(Role).filter(Role.role_name == role_name).first():
            db.add(Role(role_name=role_name))
    db.commit()


def create_admin(email: str, password: str, full_name: str | None = None) -> None:
    Base.metadata.create_all(bind=engine)
    db = SessionLocal()
    try:
        ensure_roles(db)

        if db.query(User).filter(User.email == email).first():
            print(f"User {email} already exists.")
            return

        admin_role = db.query(Role).filter(Role.role_name == ADMIN).first()
        user = User(
            email=email,
            hashed_password=hash_password(password),
            full_name=full_name,
            role_id=admin_role.role_id,
        )
        db.add(user)
        db.commit()
        print(f"Admin user {email} created.")
    finally:
        db.close()


if __name__ == "__main__":
    if len(sys.argv) < 3:
        print("Usage: python scripts/create_admin.py <email> <password> [full_name]")
        sys.exit(1)

    create_admin(sys.argv[1], sys.argv[2], sys.argv[3] if len(sys.argv) > 3 else None)

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\scripts\seed_data.py
"""Seeds sample data for local development per the "Sample Data" section of
counselor_agent_instructions.md. Safe to re-run; skips tables that already have rows.
"""
import random
import sys
from datetime import datetime
from pathlib import Path

BACKEND_DIR = Path(__file__).resolve().parent.parent / "backend"
sys.path.insert(0, str(BACKEND_DIR))

from database.base import Base  # noqa: E402
from database.session import SessionLocal, engine  # noqa: E402
from models import (  # noqa: E402
    AcademicHistory,
    BehavioralMetric,
    Certification,
    DiscoveryQuestion,
    LearningPlatformData,
    Project,
    Role,
    Skill,
    Student,
    StudentSkill,
    User,
)
from security.password import hash_password  # noqa: E402
from security.roles import ALL_ROLES, STUDENT  # noqa: E402

# Local dev login accounts (seeded only once). Password for all: "Password123!"
DEMO_PASSWORD = "Password123!"
DEMO_LOGINS = [
    ("admin@example.com", "ADMIN", "Ava Admin"),
    ("counselor@example.com", "COUNSELOR", "Cara Counselor"),
    ("mentor@example.com", "MENTOR", "Milo Mentor"),
    ("readonly@example.com", "READ_ONLY", "Ray ReadOnly"),
]

TECHNICAL_SKILLS = ["Python", "Java", "SQL", "Cloud", "AI", "Data Analytics"]
SOFT_SKILLS = ["Communication", "Leadership", "Presentation", "Problem Solving", "Collaboration"]

DISCOVERY_QUESTIONS = [
    {
        "category": "Interests",
        "question_text": "What kind of work excites you?",
        "options": {
            "A": "Building software",
            "B": "Solving business problems",
            "C": "Designing products",
            "D": "Research and innovation",
            "E": "Managing teams",
        },
    },
    {
        "category": "Preferred Working Style",
        "question_text": "Do you prefer?",
        "options": {"A": "Working alone", "B": "Small teams", "C": "Large teams", "D": "Client interaction"},
    },
    {
        "category": "Strength Analysis",
        "question_text": "Which activity do you enjoy the most?",
        "options": {"A": "Coding", "B": "Mathematics", "C": "Creativity", "D": "Communication", "E": "Analysis"},
    },
    {
        "category": "Future Preferences",
        "question_text": "Would you like:",
        "options": {
            "A": "High salary",
            "B": "Work-life balance",
            "C": "Research opportunities",
            "D": "Entrepreneurship",
            "E": "Government job",
        },
    },
]

CAREER_CERTIFICATIONS = [
    ("DataScientist", "Google Professional ML Engineer", "Google"),
    ("DataScientist", "Azure AI Engineer Associate", "Microsoft"),
    ("AIEngineer", "TensorFlow Developer Certificate", "Google"),
    ("CloudArchitect", "AWS Certified Solutions Architect", "AWS"),
    ("ProductManager", "Certified Scrum Product Owner", "Scrum Alliance"),
]

CAREER_PROJECTS = [
    ("DataScientist", "Student Performance Prediction", "Predict academic outcomes from historical data", "Intermediate"),
    ("DataScientist", "Sales Dashboard", "Interactive Power BI/Tableau sales dashboard", "Beginner"),
    ("AIEngineer", "Image Classification", "CNN based image classifier", "Advanced"),
    ("AIEngineer", "AI Chatbot", "Conversational assistant using an LLM", "Intermediate"),
    ("CloudArchitect", "End-to-End AI Application", "Deploy an ML model with MLOps on the cloud", "Advanced"),
]

FIRST_NAMES = ["John", "Priya", "Aman", "Sara", "Rahul", "Neha", "Vikram", "Anita", "Karan", "Divya"]
LAST_NAMES = ["Sharma", "Verma", "Gupta", "Singh", "Kumar", "Iyer", "Nair", "Reddy", "Das", "Mehta"]
LOCATIONS = ["Delhi", "Mumbai", "Bengaluru", "Gwalior", "Pune", "Chennai"]
BOARDS = ["CBSE", "ICSE", "State Board"]
SUBJECT_SETS = [
    ["Physics", "Math", "Computer Science"],
    ["Physics", "Chemistry", "Math"],
    ["Accountancy", "Business Studies", "Economics"],
]
PLATFORMS = ["Coursera", "Udemy", "LinkedIn Learning", "YouTube Learning", "Coding Platforms"]

STUDENT_COUNT = 20


def seed() -> None:
    Base.metadata.create_all(bind=engine)
    db = SessionLocal()
    try:
        if db.query(Skill).count() == 0:
            for name in TECHNICAL_SKILLS:
                db.add(Skill(skill_name=name, skill_type="TECHNICAL"))
            for name in SOFT_SKILLS:
                db.add(Skill(skill_name=name, skill_type="SOFT"))
            db.commit()

        if db.query(DiscoveryQuestion).count() == 0:
            for question in DISCOVERY_QUESTIONS:
                db.add(DiscoveryQuestion(**question, version=1))
            db.commit()

        if db.query(Certification).count() == 0:
            for career, cert, provider in CAREER_CERTIFICATIONS:
                db.add(Certification(career_name=career, certification_name=cert, provider=provider))
            db.commit()

        if db.query(Project).count() == 0:
            for career, name, desc, level in CAREER_PROJECTS:
                db.add(Project(career_name=career, project_name=name, description=desc, difficulty_level=level))
            db.commit()

        skills = db.query(Skill).all()

        if db.query(Student).count() == 0:
            for _ in range(STUDENT_COUNT):
                student = Student(
                    name=f"{random.choice(FIRST_NAMES)} {random.choice(LAST_NAMES)}",
                    age=random.randint(16, 19),
                    education="12th Science",
                    location=random.choice(LOCATIONS),
                    board=random.choice(BOARDS),
                    subjects=random.choice(SUBJECT_SETS),
                    created_date=datetime.utcnow(),
                )
                db.add(student)
                db.flush()

                for subject in student.subjects:
                    db.add(
                        AcademicHistory(
                            student_id=student.student_id,
                            subject=subject,
                            score=random.randint(60, 99),
                            attendance=random.randint(75, 100),
                            learning_behavior=random.choice(["Consistent", "Improving", "Needs Support"]),
                            recorded_date=datetime.utcnow(),
                        )
                    )

                for platform in random.sample(PLATFORMS, k=2):
                    db.add(
                        LearningPlatformData(
                            student_id=student.student_id,
                            platform=platform,
                            courses_completed=random.randint(1, 25),
                            hours_spent=random.randint(5, 150),
                            skill_area=random.choice(TECHNICAL_SKILLS),
                            recorded_date=datetime.utcnow(),
                        )
                    )

                for skill in random.sample(skills, k=5):
                    db.add(
                        StudentSkill(
                            student_id=student.student_id,
                            skill_id=skill.skill_id,
                            proficiency_score=random.randint(30, 95),
                            assessed_date=datetime.utcnow(),
                        )
                    )

                db.add(
                    BehavioralMetric(
                        student_id=student.student_id,
                        learning_speed=random.randint(50, 95),
                        consistency=random.randint(50, 95),
                        curiosity_index=random.randint(50, 95),
                        persistence=random.randint(50, 95),
                        goal_completion_rate=random.randint(50, 95),
                        recorded_date=datetime.utcnow(),
                    )
                )

            db.commit()

        if db.query(Role).count() == 0:
            for role_name in ALL_ROLES:
                db.add(Role(role_name=role_name))
            db.commit()

        if db.query(User).count() == 0:
            roles_by_name = {r.role_name: r for r in db.query(Role).all()}
            for email, role_name, full_name in DEMO_LOGINS:
                db.add(
                    User(
                        email=email,
                        hashed_password=hash_password(DEMO_PASSWORD),
                        full_name=full_name,
                        role_id=roles_by_name[role_name].role_id,
                    )
                )
            # Link one demo STUDENT login to the first seeded student record.
            first_student = db.query(Student).order_by(Student.student_id).first()
            if first_student is not None:
                db.add(
                    User(
                        email="student@example.com",
                        hashed_password=hash_password(DEMO_PASSWORD),
                        full_name=first_student.name,
                        role_id=roles_by_name[STUDENT].role_id,
                        student_id=first_student.student_id,
                    )
                )
            db.commit()

        print(f"Seed complete: {db.query(Student).count()} students in database.")
    finally:
        db.close()


if __name__ == "__main__":
    seed()

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\.env.example
# Database
POSTGRES_USER=counselor
POSTGRES_PASSWORD=changeme
POSTGRES_DB=counselor_agent
DATABASE_URL=postgresql://counselor:changeme@db:5432/counselor_agent

# Auth
JWT_SECRET=change-me-in-production
JWT_ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=30
REFRESH_TOKEN_EXPIRE_DAYS=7

# AI
GEMINI_API_KEY=
# Active model - switch by changing this value to any entry in GEMINI_AVAILABLE_MODELS.
GEMINI_MODEL=gemini-2.5-flash
GEMINI_AVAILABLE_MODELS=gemini-2.5-flash,gemini-2.5-pro,gemini-2.0-flash,gemini-1.5-pro
GEMINI_TEMPERATURE=0.7
# Approximate word-based cap on prompt size sent to the LLM.
GEMINI_MAX_INPUT_WORDS=3000
# Hard cap on generated response length (tokens).
GEMINI_MAX_OUTPUT_TOKENS=1024

# App
ENVIRONMENT=development

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\docker-compose.yml
services:
  db:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    ports:
      - "5432:5432"
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 5s
      timeout: 5s
      retries: 10

  backend:
    build:
      context: .
      dockerfile: backend/Dockerfile
    env_file: .env
    ports:
      - "8000:8000"
    depends_on:
      db:
        condition: service_healthy

volumes:
  db_data:

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\pytest.ini
[pytest]
testpaths = tests

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\requirements.txt
fastapi==0.115.6
uvicorn[standard]==0.32.1
pydantic==2.10.4
pydantic-settings==2.7.0
email-validator==2.2.0
sqlalchemy==2.0.36
alembic==1.14.0
psycopg2-binary==2.9.10
python-jose[cryptography]==3.3.0
passlib[bcrypt]==1.7.4
python-multipart==0.0.20
pgvector==0.3.6
langchain==0.3.13
langgraph==0.2.60
langchain-google-genai==2.0.7
google-generativeai==0.8.3

# Testing
pytest==8.3.4
httpx==0.28.1

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\README.md
# Career Counselor Agent


## Architecture at a Glance

- **Backend**: Python 3.12, FastAPI, SQLAlchemy 2.0 + Alembic, PostgreSQL 16 + pgvector,
  JWT auth with role-based access control, LangGraph/LangChain orchestration over Gemini.
- **Frontend**: Angular 19, standalone components, signals, functional guards/interceptors.
  Two portals behind one app shell: the Student Portal (role `STUDENT`) and the
  Counselor/Admin Portal (roles `ADMIN`, `COUNSELOR`, `MENTOR`, `READ_ONLY`).

---

## 1. Prerequisites

| Tool | Version | Required for |
|---|---|---|
| Docker + Docker Compose | any recent | Quickest way to run backend + Postgres/pgvector |
| Python | 3.12 | Running the backend outside Docker, migrations, seed scripts |
| Node.js + npm | 18+ (tested with Node 26 / npm 11) | Frontend build/serve |
| A Gemini API key | - | Any real (non-mock) AI feature (recommendation rationale, mentor chat, embeddings) |

---

## 2. Build & Run

### 2.1 Full stack, Docker (backend + database)

```powershell
copy .env.example .env
# edit .env and set GEMINI_API_KEY if you want real AI responses (optional, see section 3)
docker-compose up --build
```

This starts:
- `db` - PostgreSQL 16 with the `pgvector` extension, port `5432`
- `backend` - FastAPI app, port `8000`

- Backend health check: http://localhost:8000/health
- Backend Swagger docs: http://localhost:8000/docs

Docker Compose does not run migrations/seed data automatically. After the containers are
healthy, run once (from the repo root, with a Python environment that has
`requirements.txt` installed, or `docker exec` into the `backend` container):

```powershell
cd backend
alembic upgrade head
python ..\scripts\seed_data.py
python ..\scripts\create_admin.py admin@example.com "ChangeMe123!" "Admin User"
```

### 2.2 Backend only, no Docker

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install -r requirements.txt
copy .env.example .env
# start a local Postgres with the pgvector extension, then:
cd backend
alembic upgrade head
python ..\scripts\seed_data.py
python ..\scripts\create_admin.py admin@example.com "ChangeMe123!" "Admin User"
uvicorn main:app --reload
```

`seed_data.py` also creates demo login accounts (password `Password123!` for all) - see
[section 4](#4-functionality-testing-guide) for the full role/URL/credential matrix.

### 2.3 Frontend, local development

```powershell
cd frontend
npm install
npm start
```

Serves at http://localhost:4200 with `useMockData: true` (see section 3.2) by default, so
the whole frontend can be exercised **without the backend running at all**.

### 2.4 Frontend, production build

```powershell
cd frontend
npm install
npm run build
```

Output is a static bundle in `frontend/dist/frontend/`. Deploy it behind any static file
host or reverse proxy (nginx, Cloud Storage + CDN, etc.) and point it at the real backend
API by setting `apiUrl` in `frontend/src/environments/environment.ts` (used for the
production build) before building, then reverse-proxy `/api/v1` to the FastAPI backend.

### 2.5 Deployment notes

- The `infrastructure/` folder (`docker/`, `terraform/`, `gcp/`) is reserved for the
  Phase 17+ cloud deployment work (Cloud Run/Cloud SQL/GCS, per
  [counselor_agent_instructions.md](counselor_agent_instructions.md)) and is not yet
  populated - today's supported deployment path is `docker-compose` for backend+db and a
  static build for the frontend, as described above.
- `backend/Dockerfile` builds a standalone `uvicorn` image suitable for any container
  runtime (Cloud Run, ECS, plain Docker) once wired to a managed Postgres+pgvector
  instance and the environment variables in section 3.

---

## 3. Configuration

All backend configuration is environment-variable driven (see `.env.example`), loaded via
`backend/config/settings.py`.

### 3.1 LLM (Gemini) model & token limits

```env
GEMINI_API_KEY=
# Active model - switch by changing this to any entry in GEMINI_AVAILABLE_MODELS.
GEMINI_MODEL=gemini-2.5-flash
GEMINI_AVAILABLE_MODELS=gemini-2.5-flash,gemini-2.5-pro,gemini-2.0-flash,gemini-1.5-pro
GEMINI_TEMPERATURE=0.7
# Approximate word-based cap on prompt size sent to the LLM (longer prompts are truncated).
GEMINI_MAX_INPUT_WORDS=3000
# Hard cap on generated response length (tokens), enforced via the SDK's generation_config.
GEMINI_MAX_OUTPUT_TOKENS=1024
```

To switch models (e.g. from `gemini-2.5-flash` to `gemini-2.5-pro`), change `GEMINI_MODEL`
and restart the backend - no code changes required. If `GEMINI_API_KEY` is unset, AI-backed
features (mentor chat replies, recommendation rationale) gracefully fall back to
deterministic, rule-based text instead of failing.

### 3.2 Frontend mock-data mode

The frontend can run entirely against local JSON fixtures instead of the real backend,
controlled by `environment.useMockData`:

| File | `useMockData` | Used by |
|---|---|---|
| `frontend/src/environments/environment.development.ts` | `true` | `npm start` / `ng serve` |
| `frontend/src/environments/environment.ts` | `false` | `npm run build` (production) |

When `true`, a `mockApiInterceptor` (`frontend/src/app/mocks/mock-api.interceptor.ts`)
serves canned data from `frontend/src/app/mocks/data/*.json` for every API route instead
of forwarding requests to the backend - see [section 4.3](#43-testing-without-a-backend-mock-mode)
to test this way.

---

## 4. Functionality Testing Guide

### 4.1 Demo accounts

`scripts/seed_data.py` creates these accounts (password **`Password123!`** for all):

| Email | Role | Notes |
|---|---|---|
| `admin@example.com` | `ADMIN` | Full access to the Counselor/Admin Portal |
| `counselor@example.com` | `COUNSELOR` | Full access to the Counselor/Admin Portal |
| `mentor@example.com` | `MENTOR` | Access to the Counselor/Admin Portal |
| `readonly@example.com` | `READ_ONLY` | Access to the Counselor/Admin Portal (view-only intent; backend enforces write restrictions) |
| `student@example.com` | `STUDENT` | Linked to the first seeded student record, for the Student Portal |

The frontend mock-data fixtures (`frontend/src/app/mocks/data/auth.json`) mirror the same
5 accounts/password so the same credentials work whether you're testing against the real
backend or in mock mode.

### 4.2 URLs

| App | URL |
|---|---|
| Frontend | http://localhost:4200 |
| Backend Swagger/OpenAPI docs | http://localhost:8000/docs |
| Backend health check | http://localhost:8000/health |

Log in at http://localhost:4200/login. Students land on `/student`; all other roles land
on `/counselor`.

### 4.3 Testing without a backend (mock mode)

```powershell
cd frontend
npm install
npm start
```

`ng serve` uses `environment.development.ts` (`useMockData: true`) automatically. Log in
with any account from the table above (password `Password123!`) - every screen renders
fixture data and CRUD actions (create/edit/delete) mutate an in-memory store for the
session, so create/update/delete flows can be exercised end-to-end with zero backend setup.

### 4.4 Student Portal walkthrough (role: `STUDENT`)

Log in as `student@example.com`. Covers: discovering career paths, identifying
strengths/weaknesses, understanding skill gaps, selecting a roadmap, certifications,
projects, college/course context, and continuous progress monitoring.

| Step | URL | What to verify |
|---|---|---|
| Dashboard | `/student` | Quick links to recommendations, roadmap, mentor chat, progress |
| Profile | `/student/profile` | Name, age, education, board, subjects (read-only; managed by counselor) |
| Academic & Learning History | `/student/academic-history` | Subject scores/attendance, learning-platform course history |
| Skill Assessment | `/student/skills` | Technical/behavioral skill proficiency + behavioral metrics (learning speed, consistency, curiosity, persistence) |
| Discovery Questions | `/student/discovery` | Answer each question in turn; answered questions stay visible; a completion message appears once the bank is exhausted |
| Career Recommendations | `/student/recommendations` | Ranked careers with match % and rationale; "View Skill Gaps" link per card |
| Skill Gap Analysis | `/student/gap-analysis` | Matched vs. missing skills per recommended career |
| Roadmap | `/student/roadmap` | Milestones with status (`COMPLETED`/`IN_PROGRESS`/`PENDING`), topics, resources/certifications, suggested project |
| Mentor Chat | `/student/mentor-chat` | Send a message; verify a reply referencing your top recommendation is appended |
| Progress Tracker | `/student/progress` | Milestones completed / total, completion %, last reassessment date |

### 4.5 Counselor/Admin Portal walkthrough (roles: `ADMIN`, `COUNSELOR`, `MENTOR`, `READ_ONLY`)

Log in as `counselor@example.com` (or any of the other three non-student roles - the
frontend route guard grants all four the same screens).

| Step | URL | What to verify |
|---|---|---|
| Dashboard | `/counselor` | Active students, assessments completed today, avg. roadmap completion, top recommended careers |
| Student Management | `/counselor/students` | Search/list students; create a new student; delete a student |
| Student Detail | `/counselor/students/:id` | Read-only aggregate view: academic history, skills, recommendations for one student |
| Question Bank | `/counselor/question-bank` | List discovery questions; create a question; edit a question and confirm its version increments |
| Career Criteria | `/counselor/career-criteria` | Read-only career definitions (required/target skills, market demand); create/edit/delete certifications and projects |
| Analytics & Reports | `/counselor/analytics` | Career demand trends, top recommended careers, roadmap completion rate, cohort skill-gap heatmap, certification uptake funnel |

### 4.6 Backend API testing (Swagger)

Open http://localhost:8000/docs, use `POST /api/v1/auth/login` with any demo account to
get a bearer token, click **Authorize** and paste it, then exercise any endpoint directly -
useful for verifying role-based 403s (e.g. `READ_ONLY` attempting a write endpoint) that
aren't distinguished in the frontend routing.

---

## 5. Running Tests

```powershell
pip install -r requirements.txt
pytest
```

Frontend unit tests:

```powershell
cd frontend
npm test
```

-----------------------------------------------------------------
C:\ws\agent\admission\counselor-agent\counselor_agent_instructions.md
# Career Counselor Agent - Complete GitHub Copilot Implementation Instructions

## Objective

Build a production-ready AI-powered **Career Counselor Agent** platform for a College/School.

Unlike a simple recommendation tool, this agent must continuously analyze a student's
profile, academic history, learning history, skills, interests, aptitude, behavior, and
real-time responses — together with live market demand — to produce a personalized,
continuously-updated career roadmap.

The solution should:

- Discover suitable career paths for each student
- Identify strengths and weaknesses
- Identify skill gaps against target careers
- Recommend a personalized learning roadmap
- Recommend certifications and projects
- Guide college/course selection
- Run a real-time (dynamic, non-static) discovery conversation instead of long forms
- Continuously track progress and re-assess as interests/market demand change
- Support future Admission Agent and Marketing Agent integration
- Be deployable on GCP
- Follow Enterprise Architecture standards
- Include sample test data
- Include Docker deployment
- Include CI/CD pipeline support

---

# Technology Stack

## Frontend

- Angular 19
- Angular CLI
- TypeScript
- Tailwind CSS
- Angular Material
- ng2-charts (Chart.js)

## Backend

- Python 3.12
- FastAPI
- Pydantic
- SQLAlchemy
- Alembic

## AI

- Gemini 2.5
- LangGraph
- LangChain
- PGVector

## Database

- PostgreSQL 16

## Authentication

- JWT
- RBAC

## Hosting

- Google Cloud Run

## Storage

- Google Cloud Storage

## CI/CD

- GitHub Actions

## Containers

- Docker
- Docker Compose

---

# Project Folder Structure

Create the project using the following structure.

counselor-agent/
|
├── README.md
├── docker-compose.yml
├── .env.example
├── requirements.txt
├── .gitignore
|
├── frontend/
│   ├── package.json
│   ├── angular.json
│   ├── tsconfig.app.json
│   ├── tsconfig.spec.json
│   ├── public/
│   ├── src/
│   │   ├── app/
│   │   │   ├── core/
│   │   │   ├── shared/
│   │   │   ├── features/
│   │   │   ├── services/
│   │   │   ├── models/
│   │   │   ├── app.routes.ts
│   │   │   └── app.config.ts
│   │   ├── assets/
│   │   ├── environments/
│   │   ├── index.html
│   │   ├── main.ts
│   │   └── styles.css
│   └── Dockerfile
|
├── backend/
│   ├── main.py
│   ├── Dockerfile
│   │
│   ├── api/
│   │   ├── routes/
│   │   └── dependencies/
│   │
│   ├── agents/
│   │   ├── profile_agent/
│   │   ├── learning_agent/
│   │   ├── assessment_agent/
│   │   ├── recommendation_agent/
│   │   ├── roadmap_agent/
│   │   ├── mentor_agent/
│   │   ├── progress_tracking_agent/
│   │   └── orchestrator/
│   │
│   ├── services/
│   ├── repositories/
│   ├── database/
│   ├── models/
│   ├── schemas/
│   ├── prompts/
│   ├── workflows/
│   ├── middleware/
│   ├── security/
│   ├── config/
│   └── utils/
|
├── infrastructure/
│   ├── terraform/
│   ├── gcp/
│   └── docker/
|
├── scripts/
│   ├── seed_data.py
│   ├── create_admin.py
│   └── generate_test_data.py
|
├── docs/
│   ├── architecture.md
│   ├── api-specification.md
│   ├── deployment-guide.md
│   └── database-design.md
|
└── tests/
    ├── unit/
    ├── integration/
    ├── api/
    └── e2e/

---

# Standalone System Requirement

The Counselor Agent MUST be built and runnable as a fully independent system.

- Do not import code from, or share a database/runtime with, `marketing-agent` or
  `college-admission-agent`.
- No module in this project may block on the Marketing Agent or Admission Agent being
  present.
- Future integration with the Admission Agent, Marketing Agent (or any other future
  agent) must happen only through versioned REST APIs (`/api/v1/...`) and documented
  contracts in `docs/api-specification.md` — never through shared code, shared tables,
  or direct imports.
- The system must start, seed data, and pass its own test suite using only
  `docker-compose up` with no other project present on disk.

---

# Implementation Roadmap (Ordered Build Steps With Dependencies)

Build in the phase order below. Each phase lists what it depends on and what it
must deliver before the next dependent phase starts. Modules explicitly marked
"parallel-safe" have no dependency on each other and may be built in any order
or concurrently once their listed dependencies are satisfied.

## Phase 0 - Project Scaffolding

Depends on: nothing (first step)

Deliverables:

- Repo skeleton per "Project Folder Structure" section
- `.env.example`, `requirements.txt`, `docker-compose.yml` (postgres + pgvector only for now)
- Empty FastAPI app (`backend/main.py`) with a `/health` endpoint
- Empty Angular workspace (`frontend/`) that builds and serves a placeholder page

Exit criteria: `docker-compose up` starts backend + db, `GET /health` returns 200.

---

## Phase 1 - Database Schema & Migrations

Depends on: Phase 0

Deliverables:

- SQLAlchemy models for all tables in "Database Design" (students, academic_history,
  learning_platform_data, skills, student_skills, behavioral_metrics,
  discovery_questions, discovery_responses, assessment_scores, career_recommendations,
  skill_gaps, roadmaps, roadmap_milestones, certifications, projects,
  mentor_conversations, progress_tracking, prompts)
- Explicit foreign keys: `academic_history.student_id → students.student_id`,
  `student_skills.student_id → students.student_id`,
  `student_skills.skill_id → skills.skill_id`,
  `assessment_scores.student_id → students.student_id`,
  `career_recommendations.student_id → students.student_id`,
  `skill_gaps.recommendation_id → career_recommendations.recommendation_id`,
  `roadmaps.student_id → students.student_id` and
  `roadmaps.recommendation_id → career_recommendations.recommendation_id`,
  `roadmap_milestones.roadmap_id → roadmaps.roadmap_id`,
  `progress_tracking.student_id → students.student_id`
- Alembic initialized, first migration generated and applied
- `scripts/seed_data.py` creates the sample data volumes listed in "Sample Data"

Exit criteria: `alembic upgrade head` + `python scripts/seed_data.py` succeed against a
fresh Postgres instance.

---

## Phase 2 - Authentication & RBAC (Module 1)

Depends on: Phase 1 (needs `users`/`roles` tables added to the schema)

Deliverables:

- `users` and `roles` tables (add to Phase 1 schema if not already present)
- JWT login/refresh/logout/password-reset endpoints
- RBAC middleware enforcing the 5 roles (ADMIN, COUNSELOR, MENTOR, STUDENT, READ_ONLY)
- `scripts/create_admin.py`

Exit criteria: protected route returns 401/403 correctly per role; admin user can log in.

---

## Phase 3 - Student Profile Management (Module 2)

Depends on: Phase 2 (all CRUD routes must be RBAC-protected)

Deliverables: CRUD APIs for Student Profile (name, age, education, location, board,
subjects) per "Data Inputs → Student Profile".

Exit criteria: full CRUD + Swagger docs for this module.

---

## Phase 4 - AI Infrastructure (Gemini + PGVector + LangGraph skeleton)

Depends on: Phase 1 (DB must exist for PGVector extension + embeddings tables)

Deliverables:

- `services/gemini_service.py` wrapper (prompt in, text out, error handling, retries)
- PGVector extension enabled; embedding functions for Careers, Certifications,
  Courses, Projects, Job Market Roles, FAQs
- `agents/orchestrator/` LangGraph `StateGraph` skeleton with a single passthrough
  node, memory, and retry/error-handling wired but no business nodes yet
- `prompts` table CRUD + prompt loader used by all future agents (see "LLM Prompt
  Strategy")

Exit criteria: a sample prompt round-trips through Gemini and a response is logged.

---

## Phase 5 - Academic & Learning History Agent (Module 3)

Depends on: Phase 3 (student profile must exist), Phase 4 (Gemini/PGVector/orchestrator)

Deliverables:

- Academic history ingestion API (grades, semester scores, subject performance,
  attendance, learning behavior)
- Learning platform integration adapters (Coursera, Udemy, LinkedIn Learning, YouTube
  Learning, Coding Platforms) capturing courses completed and hours per skill area
  (`python_hours`, `ai_hours`, `cloud_hours`, etc.)

Exit criteria: academic + learning history for a student persists and is retrievable
via `GET /api/v1/students/{id}/history`.

---

## Phase 6 - Skill Assessment Agent (Module 4)

Depends on: Phase 5 (needs learning/academic history as an assessment input)

Deliverables: technical skill catalog (Python, Java, SQL, Cloud, AI, Data Analytics),
soft skill catalog (Communication, Leadership, Presentation, Problem Solving,
Collaboration), and behavioral analysis capture (learning speed, consistency,
curiosity index, persistence, goal completion rate).

Parallel-safe with: Phase 7 (Discovery Q&A Engine).

Exit criteria: `student_skills` and `behavioral_metrics` populated and scorable.

---

## Phase 7 - Real-Time Career Discovery Q&A Engine (Module 5)

Depends on: Phase 4 (orchestrator + Gemini), Phase 3 (student profile)

Deliverables: dynamic (non-static) question engine covering Interests, Preferred
Working Style, Strength Analysis, and Future Preferences (see "Real-Time Career
Discovery Questions"); persists responses to `discovery_responses`.

Parallel-safe with: Phase 6.

Exit criteria: a full discovery session for a student produces stored, scorable
answers.

---

## Phase 8 - AI Assessment Engine / Dimension Scoring (Module 6)

Depends on: Phase 6 (skills/behavior), Phase 7 (discovery responses) — also reads
academic history from Phase 5

Deliverables: weighted dimension scoring engine (Interest 30%, Skill 25%, Academic
History 15%, Aptitude 15%, Market Demand 15%) producing a per-career match score
persisted to `assessment_scores`.

Exit criteria: scoring engine returns a ranked score map (e.g.
`{"SoftwareEngineer": 92, "DataScientist": 89, ...}`) for a seeded student.

---

## Phase 9 - Career Recommendation Engine (Module 7)

Depends on: Phase 8 (dimension scores must exist)

Deliverables: rule + LLM based recommendation engine evaluating candidate careers
(AI/ML Engineer, Data Scientist, Cloud Architect, Product Manager, etc.) against
their criteria and persisting ranked results to `career_recommendations`.

Exit criteria: `POST /api/v1/recommendations/generate` returns ranked careers with
match scores and rationale.

---

## Phase 10 - Gap Identification Engine (Module 8)

Depends on: Phase 9 (needs a target/recommended career to diff against)

Deliverables: engine comparing a student's current skills against a target career's
required skills, producing a matched/missing skill list persisted to `skill_gaps`.

Exit criteria: gap analysis for a seeded student + target career returns correct
matched (✓) and missing (✗) skills.

---

## Phase 11 - Roadmap Generator Agent (Module 9)

Depends on: Phase 10 (skill gaps drive roadmap content)

Deliverables: multi-month roadmap generator (learning topics, resources,
certifications, projects per phase) persisted to `roadmaps` and `roadmap_milestones`,
per the "Personalized Roadmap Generator" example.

Parallel-safe with: Phase 12 (Mentor Agent).

Exit criteria: `POST /api/v1/roadmap/generate` returns a full month-by-month roadmap
for a target career.

---

## Phase 12 - Mentor Agent (Module 10)

Depends on: Phase 4 (orchestrator + Gemini), Phase 9 (recommendations provide
conversation context)

Deliverables: real-time conversational agent (chat) that explains recommendations,
answers student questions, and can trigger re-assessment; persists sessions to
`mentor_conversations`.

Parallel-safe with: Phase 11.

Exit criteria: a chat round-trip references the student's actual recommendations and
gaps in its answer.

---

## Phase 13 - Progress Tracking & Continuous Reassessment Agent (Module 11)

Depends on: Phase 11 (roadmap must exist to track progress against), Phase 6 (skill
scores must be re-computable)

Deliverables: milestone completion tracking, automatic re-scoring trigger when new
learning/skill/behavioral data arrives, and roadmap adjustment logic; persists to
`progress_tracking`.

Exit criteria: completing a milestone updates progress % and can trigger Phase 8/9
re-run.

---

## Phase 14 - Market Trend & Analytics Agent (Module 12)

Depends on: Phase 9 (recommendations), Phase 13 (progress data) — needs both to
compute demand-adjusted rankings and cohort dashboards

Deliverables: market demand data source integration (Job Market MCP) feeding the
Phase 8 scoring weight, plus dashboard metrics endpoints and chart data (bar, line,
pie, funnel) across the student cohort.

---

## Phase 15 - Full Multi-Agent Orchestration

Depends on: Phases 5-14 (every sub-agent must exist as a callable LangGraph node)

Deliverables: wire Profile/Learning/Assessment/Recommendation/Roadmap/Mentor/
Progress-Tracking agents as nodes into the Phase 4 orchestrator skeleton; implement
the fan-out/fan-in graph shown in "Agent Architecture", with retry logic and error
handling active on every node.

---

## Phase 16 - Frontend (Angular)

Depends on: incrementally on each backend phase's APIs (start after Phase 2; add
screens as each module's API becomes available). Student Portal screens need Phases
3, 5-9, 11, 12; Counselor/Admin Portal screens need Phases 2, 13, 14.

Deliverables: Student Portal and Counselor/Admin Portal pages per "Frontend Screens".

---

## Phase 17 - Security Hardening

Depends on: all API modules existing (Phases 2-14)

Deliverables: rate limiting, input validation audit, CORS policy, SQLi/XSS
verification pass across every endpoint.

---

## Phase 18 - Testing

Depends on: continuous — unit tests per phase as it's built; integration tests once
Phases 1-15 exist; e2e tests (Discovery-to-Recommendation Flow, Roadmap Generation
Flow, Progress Tracking Flow) only after Phase 15 (orchestration) and Phase 16
(frontend) are complete.

Exit criteria: coverage > 80% before Phase 20 (CI/CD) is finalized.

---

## Phase 19 - DevOps Finalization

Depends on: Phase 0 (base compose file), extended once frontend (16) and all backend
services exist.

Deliverables: final `docker-compose.yml` with frontend, backend, postgres, pgvector
services; both Dockerfiles production-ready.

---

## Phase 20 - CI/CD (GitHub Actions)

Depends on: Phase 18 (tests must exist to run in the pipeline), Phase 19 (Docker
build step needs final Dockerfiles).

Deliverables: pipeline stages Build → Test → Sonar Scan → Security Scan →
Docker Build → Deploy.

---

## Phase 21 - GCP Deployment

Depends on: Phase 20 (pipeline must be green before first deploy)

Deliverables: Terraform for Cloud Run services (counselor-frontend,
counselor-backend), Cloud SQL, GCS bucket, Vertex AI Gemini wiring; deployment
guide in `docs/deployment-guide.md`.

---

## Dependency Graph

```mermaid
graph TD
    P0[Phase 0: Scaffolding] --> P1[Phase 1: DB Schema]
    P1 --> P2[Phase 2: Auth/RBAC]
    P2 --> P3[Phase 3: Student Profile]
    P1 --> P4[Phase 4: AI Infra]
    P3 --> P5[Phase 5: Academic/Learning History]
    P4 --> P5
    P5 --> P6[Phase 6: Skill Assessment]
    P4 --> P7[Phase 7: Discovery Q&A]
    P3 --> P7
    P6 --> P8[Phase 8: Dimension Scoring]
    P7 --> P8
    P8 --> P9[Phase 9: Career Recommendation]
    P9 --> P10[Phase 10: Gap Identification]
    P10 --> P11[Phase 11: Roadmap Generator]
    P4 --> P12[Phase 12: Mentor Agent]
    P9 --> P12
    P11 --> P13[Phase 13: Progress Tracking]
    P6 --> P13
    P9 --> P14[Phase 14: Market Trend/Analytics]
    P13 --> P14
    P5 --> P15[Phase 15: Full Orchestration]
    P6 --> P15
    P7 --> P15
    P8 --> P15
    P9 --> P15
    P10 --> P15
    P11 --> P15
    P12 --> P15
    P13 --> P15
    P14 --> P15
    P2 --> P16[Phase 16: Frontend]
    P15 --> P17[Phase 17: Security Hardening]
    P15 --> P18[Phase 18: Testing]
    P16 --> P18
    P0 --> P19[Phase 19: DevOps]
    P16 --> P19
    P18 --> P20[Phase 20: CI/CD]
    P19 --> P20
    P20 --> P21[Phase 21: GCP Deployment]
```

---

# Core Business Modules

Implement the following modules.

---

## Module 1 - Authentication

Features:

- Login
- Logout
- Refresh Token
- Password Reset
- Role Based Access Control

Roles:

- ADMIN
- COUNSELOR
- MENTOR
- STUDENT
- READ_ONLY

---

## Module 2 - Student Profile Management

Capture:

```json
{
  "name": "John",
  "age": 18,
  "education": "12th Science",
  "location": "Delhi",
  "board": "CBSE",
  "subjects": [
    "Physics",
    "Math",
    "Computer Science"
  ]
}
```

CRUD APIs required.

---

## Module 3 - Academic & Learning History Agent

Purpose:

Ingest and analyze academic performance and learning platform activity.

### Academic History

Collect school grades, semester scores, subject performance, attendance, and
learning behavior.

```json
{
  "math": 92,
  "physics": 89,
  "english": 70,
  "computer_science": 95
}
```

### Learning Platforms Data

Integrate with:

- Coursera
- Udemy
- LinkedIn Learning
- YouTube Learning
- Coding Platforms

Capture:

```json
{
  "courses_completed": 25,
  "python_hours": 140,
  "ai_hours": 50,
  "cloud_hours": 20
}
```

API:

```
POST /api/v1/students/{id}/history
GET  /api/v1/students/{id}/history
```

---

## Module 4 - Skill Assessment Agent

### Technical Skills

- Python
- Java
- SQL
- Cloud
- AI
- Data Analytics

### Soft Skills

- Communication
- Leadership
- Presentation
- Problem Solving
- Collaboration

### Behavioral Analysis

Monitor:

- Learning speed
- Consistency
- Curiosity index
- Persistence
- Goal completion rate

API:

```
POST /api/v1/students/{id}/skills/assess
GET  /api/v1/students/{id}/skills
```

---

## Module 5 - Real-Time Career Discovery Engine

Purpose:

Ask dynamic, adaptive questions instead of a long static questionnaire.

### Interests

What kind of work excites you?

- A. Building software
- B. Solving business problems
- C. Designing products
- D. Research and innovation
- E. Managing teams

### Preferred Working Style

Do you prefer?

- A. Working alone
- B. Small teams
- C. Large teams
- D. Client interaction

### Strength Analysis

Which activity do you enjoy the most?

- A. Coding
- B. Mathematics
- C. Creativity
- D. Communication
- E. Analysis

### Future Preferences

Would you like:

- A. High salary
- B. Work-life balance
- C. Research opportunities
- D. Entrepreneurship
- E. Government job

API:

```
GET  /api/v1/discovery/next-question?studentId={id}
POST /api/v1/discovery/answer
```

---

## Module 6 - AI Assessment Engine (Dimension Scoring)

Calculate a weighted score across dimensions:

| Category         | Weight |
|-------------------|--------|
| Interest           | 30%    |
| Skill              | 25%    |
| Academic History   | 15%    |
| Aptitude           | 15%    |
| Market Demand      | 15%    |

Example output:

```json
{
  "SoftwareEngineer": 92,
  "DataScientist": 89,
  "CloudArchitect": 87,
  "ProductManager": 70
}
```

API:

```
POST /api/v1/assessment/score
```

---

## Module 7 - Career Recommendation Engine

### AI/ML Engineer

Criteria: Strong Math, Python Knowledge, Analytical Thinking, Interest in AI

### Data Scientist

Criteria: Statistics, SQL, Python, Visualization

### Cloud Architect

Criteria: Infrastructure Interest, Problem Solving, System Design

### Product Manager

Criteria: Leadership, Communication, Business Understanding

API:

```
POST /api/v1/recommendations/generate
```

---

## Module 8 - Gap Identification Engine

Example:

Target Career: **Data Scientist**

Current Skills: ✓ Python, ✓ SQL

Missing Skills: ✗ Statistics, ✗ Machine Learning, ✗ Data Visualization,
✗ Cloud Analytics

API:

```
POST /api/v1/gap-analysis
```

---

## Module 9 - Roadmap Generator Agent

### Example Roadmap

Goal: **Become Data Scientist** — Duration: **12 Months**

**Month 1-2** — Learn: Python Advanced, NumPy, Pandas, Statistics
Resources: Coursera, Kaggle, YouTube

**Month 3-4** — Learn: Machine Learning, Scikit Learn, Feature Engineering
Project: Student Performance Prediction

**Month 5-6** — Learn: SQL Advanced, Power BI, Tableau
Project: Sales Dashboard

**Month 7-9** — Learn: Deep Learning, TensorFlow, PyTorch
Project: Image Classification

**Month 10-12** — Learn: MLOps, Cloud Deployment, GitHub Portfolio
Project: End-to-End AI Application

API:

```
POST /api/v1/roadmap/generate
GET  /api/v1/roadmap/{studentId}
```

---

## Module 10 - Mentor Agent

Purpose:

Real-time conversational agent that explains recommendations, answers student
questions, and can request re-assessment.

API:

```
POST /api/v1/mentor/chat
```

Request:

```json
{
  "studentId": 1,
  "message": "Why is Data Scientist a good fit for me?"
}
```

Response:

```json
{
  "reply": "Based on your strong Math and Python scores...",
  "relatedCareer": "DataScientist"
}
```

---

## Module 11 - Progress Tracking Agent

Features:

- Milestone completion tracking
- Continuous reassessment trigger when new learning/skill/behavioral data arrives
- Roadmap adjustment based on changing interests and market demand

API:

```
POST /api/v1/progress/milestone/complete
GET  /api/v1/progress/{studentId}
```

---

## Module 12 - Market Trend & Analytics Agent

Dashboard Metrics:

- Career Demand Trends
- Cohort Skill Gap Heatmap
- Roadmap Completion Rate
- Top Recommended Careers
- Certification Uptake

Charts:

- Bar Chart
- Line Chart
- Pie Chart
- Funnel Chart

---

# Agent Architecture (Multi-Agent Design)

```
Career Counselor Agent
        │
 ┌──────┼─────────┐
 ▼      ▼         ▼
Profile Learning Assessment
Agent   Agent     Agent
 │
 ▼
Recommendation Agent
 │
 ▼
Roadmap Agent
 │
 ▼
Mentor Agent
 │
 ▼
Progress Tracking Agent
```

Implement:

- StateGraph
- Memory
- Retry Logic
- Error Handling

---

# LLM Prompt Strategy

### System Prompt

You are an expert Career Counselor AI.

Analyze:

1. Student profile
2. Academic history
3. Learning history
4. Skills
5. Interests
6. Career goals

Perform:

1. Career fit analysis
2. Skill gap identification
3. Career ranking
4. Learning roadmap creation
5. Certification recommendation
6. Project recommendation

Always explain:

- Why a career suits the student
- Required skills
- Gap analysis
- Market demand
- Expected timeline

---

# MCP/Agentic AI Tools Integration

The Career Counselor Agent can use tools such as:

- Profile Service MCP
- Learning History MCP
- Skill Assessment MCP
- Market Trend MCP
- Course Recommendation MCP
- Certification MCP
- Job Market MCP
- Roadmap Planner MCP

---

# Final Response Example

```json
{
  "recommended_careers": [
    {
      "career": "AI Engineer",
      "match_score": 94
    },
    {
      "career": "Data Scientist",
      "match_score": 91
    }
  ],
  "strengths": [
    "Mathematics",
    "Python",
    "Analytical Thinking"
  ],
  "gaps": [
    "Machine Learning",
    "Statistics",
    "Cloud AI"
  ],
  "roadmap_duration": "12 months",
  "recommended_certifications": [
    "Google Professional ML Engineer",
    "Azure AI Engineer Associate"
  ],
  "recommended_projects": [
    "Recommendation System",
    "AI Chatbot",
    "Forecasting Application"
  ]
}
```

---

# Database Design

## TABLE students

Fields:

- student_id
- name
- age
- education
- location
- board
- subjects
- created_date

---

## TABLE academic_history

Fields:

- history_id
- student_id
- subject
- score
- attendance
- learning_behavior
- recorded_date

---

## TABLE learning_platform_data

Fields:

- record_id
- student_id
- platform
- courses_completed
- hours_spent
- skill_area
- recorded_date

---

## TABLE skills

Fields:

- skill_id
- skill_name
- skill_type (TECHNICAL / SOFT)

---

## TABLE student_skills

Fields:

- id
- student_id
- skill_id
- proficiency_score
- assessed_date

---

## TABLE behavioral_metrics

Fields:

- metric_id
- student_id
- learning_speed
- consistency
- curiosity_index
- persistence
- goal_completion_rate
- recorded_date

---

## TABLE discovery_questions

Fields:

- question_id
- category
- question_text
- options
- version

---

## TABLE discovery_responses

Fields:

- response_id
- student_id
- question_id
- selected_option
- answered_date

---

## TABLE assessment_scores

Fields:

- score_id
- student_id
- career_name
- interest_score
- skill_score
- academic_score
- aptitude_score
- market_demand_score
- final_score
- computed_date

---

## TABLE career_recommendations

Fields:

- recommendation_id
- student_id
- career_name
- match_score
- rationale
- created_date

---

## TABLE skill_gaps

Fields:

- gap_id
- recommendation_id
- skill_id
- status (MATCHED / MISSING)

---

## TABLE roadmaps

Fields:

- roadmap_id
- student_id
- recommendation_id
- goal
- duration_months
- status
- created_date

---

## TABLE roadmap_milestones

Fields:

- milestone_id
- roadmap_id
- month_range
- topics
- resources
- project
- status
- completed_date

---

## TABLE certifications

Fields:

- certification_id
- career_name
- certification_name
- provider

---

## TABLE projects

Fields:

- project_id
- career_name
- project_name
- description
- difficulty_level

---

## TABLE mentor_conversations

Fields:

- conversation_id
- student_id
- message
- reply
- related_career
- created_date

---

## TABLE progress_tracking

Fields:

- tracking_id
- student_id
- roadmap_id
- milestones_completed
- completion_percentage
- last_reassessed_date

---

## TABLE prompts

Fields:

- prompt_id
- agent_name
- prompt_type
- prompt_text
- version

---

# Vector Search

Implement PGVector.

Create embeddings for:

- Careers
- Certifications
- Courses
- Projects
- Job Market Roles
- FAQs

Functions:

- Store Embeddings
- Semantic Search
- Similarity Search

---

# Sample Data

Generate test data:

## Students

100 Sample Students (with profile, academic history, learning history)

## Skills

30 Sample Skills (Technical + Soft)

## Discovery Questions

20 Sample Discovery Questions across Interests, Working Style, Strength, Future
Preferences

## Career Recommendations

10 Sample Career Definitions with Criteria (AI/ML Engineer, Data Scientist, Cloud
Architect, Product Manager, etc.)

## Roadmaps

50 Sample Generated Roadmaps

## Certifications

30 Sample Certifications

## Projects

40 Sample Projects

---

# Frontend Screens

Implement responsive pages.

## Student Portal

- Dashboard
- Profile
- Academic & Learning History
- Discovery Questions (Real-Time Chat-Style)
- Career Recommendations
- Skill Gap Analysis
- Roadmap
- Mentor Chat
- Progress Tracker

---

## Counselor/Admin Portal

### Dashboard

Widgets:

- Active Students
- Assessments Completed Today
- Top Recommended Careers
- Roadmap Completion Rate

### Student Management

- Search
- Filter
- View Profile & History

### Question Bank Management

- Create
- Update
- Version Discovery Questions

### Career Criteria Management

- Define/Update Career Criteria
- Manage Certifications & Projects Catalog

### Analytics

- Reports
- Charts
- Export

---

# API Standards

Use:

- REST APIs
- OpenAPI Swagger
- Versioned APIs

Pattern:

/api/v1/

Examples:

```
GET  /api/v1/students/{id}
POST /api/v1/discovery/answer
POST /api/v1/assessment/score
POST /api/v1/recommendations/generate
POST /api/v1/gap-analysis
POST /api/v1/roadmap/generate
POST /api/v1/mentor/chat
GET  /api/v1/progress/{studentId}
```

---

# Logging

Implement structured logging.

Log:

- Request
- Response
- Agent Execution
- Prompt Usage
- Errors

Use:

- Python Logging
- JSON Logging

---

# Security

Implement:

- JWT Authentication
- RBAC
- CORS
- Input Validation
- Rate Limiting
- SQL Injection Protection
- XSS Protection

---

# Testing

Create:

## Unit Tests

Coverage > 80%

## Integration Tests

- Database
- Agent
- API

## E2E Tests

- Discovery-to-Recommendation Flow
- Roadmap Generation Flow
- Progress Tracking Flow

---

# DevOps

Dockerize everything.

Create:

- Dockerfile
- docker-compose.yml

Services:

- frontend
- backend
- postgres
- pgvector

Commands:

```
docker-compose up
docker-compose down
```

---

# GitHub Actions

Implement pipeline.

Stages:

1. Build
2. Test
3. Sonar Scan
4. Security Scan
5. Docker Build
6. Deploy

---

# Deployment

Target Platform:

Google Cloud Run

Services:

- counselor-frontend
- counselor-backend
- postgres-cloudsql
- storage-gcs
- Vertex AI Gemini

Provide complete deployment scripts.

---

# Acceptance Criteria

The generated solution must:

1. Run locally using Docker Compose.
2. Run on GCP Cloud Run.
3. Support 10,000+ students.
4. Support multi-agent execution using LangGraph.
5. Include complete Swagger documentation.
6. Include sample data.
7. Include automated tests.
8. Include CI/CD pipeline.
9. Include role-based access.
10. Continuously re-assess and adapt recommendations as new data arrives.
11. Be production-ready and maintainable.

---

# Future Integration

Design APIs and architecture to support:

- College Admission Agent
- Marketing Agent
- Fee Collection Agent
- Student Helpdesk Agent
- Parent Agent
- Placement Agent
- Chairman Dashboard Agent

without major refactoring.

Follow Domain Driven Design (DDD), Clean Architecture, SOLID Principles, Repository
Pattern, Service Layer Pattern, and Enterprise Coding Standards.
