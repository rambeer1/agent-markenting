C:\ws\agent\website-college\backend-python\app\routers\__init__.py
C:\ws\agent\website-college\backend-python\app\routers\admission.py
"""Mirrors controller/AdmissionInquiryController.java (Module 7 - Admission
Inquiry, Module 12 - Admission Inquiry Management)."""
from fastapi import APIRouter, Depends, Query, Response, status
from sqlalchemy.orm import Session

from ..database import get_db
from ..responses import page, success
from ..schemas import AdmissionInquiryRequest, AdmissionInquiryStatusUpdateRequest
from ..security import get_current_user, require_admin_or_editor, require_any_role
from ..services import admission_service

router = APIRouter(tags=["Admission Inquiry"])


@router.post("/api/v1/admissions/inquiry", status_code=status.HTTP_201_CREATED)
def submit_inquiry(request: AdmissionInquiryRequest, db: Session = Depends(get_db)):
    return success(
        admission_service.submit_inquiry(db, request),
        "Your admission inquiry has been submitted successfully",
    )


@router.get("/api/v1/admissions/inquiry/{inquiry_id}", dependencies=[Depends(get_current_user)])
def get_inquiry_by_id(inquiry_id: int, db: Session = Depends(get_db)):
    return success(admission_service.get_inquiry_by_id(db, inquiry_id))


@router.get("/api/v1/admin/admissions", dependencies=[Depends(require_any_role)])
def get_all_inquiries(
    page_number: int = Query(0, alias="page", ge=0),
    size: int = Query(10, ge=1, le=200),
    db: Session = Depends(get_db),
):
    items, total = admission_service.get_all_inquiries(db, page_number, size)
    return success(page(items, total, page_number, size))


@router.get("/api/v1/admin/admissions/search", dependencies=[Depends(require_any_role)])
def search_inquiries(
    keyword: str,
    page_number: int = Query(0, alias="page", ge=0),
    size: int = Query(10, ge=1, le=200),
    db: Session = Depends(get_db),
):
    items, total = admission_service.search_inquiries(db, keyword, page_number, size)
    return success(page(items, total, page_number, size))


@router.get("/api/v1/admin/admissions/status/{inquiry_status}", dependencies=[Depends(require_any_role)])
def get_inquiries_by_status(
    inquiry_status: str,
    page_number: int = Query(0, alias="page", ge=0),
    size: int = Query(10, ge=1, le=200),
    db: Session = Depends(get_db),
):
    items, total = admission_service.get_inquiries_by_status(db, inquiry_status, page_number, size)
    return success(page(items, total, page_number, size))


@router.put("/api/v1/admin/admissions/{inquiry_id}/status", dependencies=[Depends(require_admin_or_editor)])
def update_status(inquiry_id: int, request: AdmissionInquiryStatusUpdateRequest, db: Session = Depends(get_db)):
    return success(admission_service.update_status(db, inquiry_id, request), "Inquiry status updated successfully")


@router.get("/api/v1/admin/admissions/export", dependencies=[Depends(require_admin_or_editor)])
def export_csv(db: Session = Depends(get_db)):
    csv_content = admission_service.export_all_as_csv(db)
    return Response(
        content=csv_content,
        media_type="text/csv",
        headers={"Content-Disposition": "attachment; filename=admission-inquiries.csv"},
    )
-------------------------------------------------------------
C:\ws\agent\website-college\backend-python\app\routers\auth.py
"""Mirrors controller/AuthController.java (Module 8 - Authentication)."""
from fastapi import APIRouter, Depends
from sqlalchemy.orm import Session

from ..database import get_db
from ..responses import success
from ..schemas import ForgotPasswordRequest, LoginRequest, RefreshTokenRequest, ResetPasswordRequest
from ..services import auth_service

router = APIRouter(prefix="/api/v1/auth", tags=["Authentication"])


@router.post("/login")
def login(request: LoginRequest, db: Session = Depends(get_db)):
    return success(auth_service.login(db, request), "Login successful")


@router.post("/refresh")
def refresh(request: RefreshTokenRequest, db: Session = Depends(get_db)):
    return success(auth_service.refresh_token(db, request), "Token refreshed successfully")


@router.post("/forgot-password")
def forgot_password(request: ForgotPasswordRequest, db: Session = Depends(get_db)):
    auth_service.forgot_password(db, request)
    return success(None, "If the account exists, password reset instructions have been sent")


@router.post("/reset-password")
def reset_password(request: ResetPasswordRequest, db: Session = Depends(get_db)):
    auth_service.reset_password(db, request)
    return success(None, "Password has been reset successfully")


@router.post("/logout")
def logout():
    # Stateless JWT: the client discards the token. Nothing to invalidate server-side.
    return success(None, "Logout successful")

-------------------------------------------------------------
C:\ws\agent\website-college\backend-python\app\routers\college.py
"""Mirrors controller/CollegeController.java (Module 2 - About College)."""
from fastapi import APIRouter, Depends
from sqlalchemy.orm import Session

from ..database import get_db
from ..responses import success
from ..services import college_service

router = APIRouter(prefix="/api/v1/college", tags=["College"])


@router.get("")
def get_college(db: Session = Depends(get_db)):
    return success(college_service.get_college_info(db))

-------------------------------------------------------------
C:\ws\agent\website-college\backend-python\app\routers\contact.py
"""Mirrors controller/ContactController.java (Module 6 - Contact Us)."""
from fastapi import APIRouter, Depends, Query, status
from sqlalchemy.orm import Session

from ..database import get_db
from ..responses import page, success
from ..schemas import ContactUsRequest
from ..security import require_any_role
from ..services import contact_service

router = APIRouter(tags=["Contact Us"])


@router.post("/api/v1/contact", status_code=status.HTTP_201_CREATED)
def submit_contact(request: ContactUsRequest, db: Session = Depends(get_db)):
    return success(contact_service.submit_contact(db, request), "Your message has been submitted successfully")


@router.get("/api/v1/admin/contacts", dependencies=[Depends(require_any_role)])
def get_all_contacts(
    page_number: int = Query(0, alias="page", ge=0),
    size: int = Query(10, ge=1, le=200),
    db: Session = Depends(get_db),
):
    items, total = contact_service.get_all_contacts(db, page_number, size)
    return success(page(items, total, page_number, size))


@router.get("/api/v1/admin/contacts/{contact_id}", dependencies=[Depends(require_any_role)])
def get_contact_by_id(contact_id: int, db: Session = Depends(get_db)):
    return success(contact_service.get_contact_by_id(db, contact_id))

-------------------------------------------------------------
C:\ws\agent\website-college\backend-python\app\routers\course.py
"""Mirrors controller/CourseController.java (Module 3 - Courses, Module 10 - Course Management)."""
from fastapi import APIRouter, Depends, Query, Response, status
from sqlalchemy.orm import Session

from ..constants import Roles
from ..database import get_db
from ..responses import page, success
from ..schemas import CourseRequest
from ..security import require_admin, require_admin_or_editor, require_any_role
from ..services import course_service

router = APIRouter(tags=["Courses"])


@router.get("/api/v1/courses")
def get_all_courses(
    page_number: int = Query(0, alias="page", ge=0),
    size: int = Query(10, ge=1, le=200),
    db: Session = Depends(get_db),
):
    items, total = course_service.get_all_courses(db, page_number, size)
    return success(page(items, total, page_number, size))


@router.get("/api/v1/courses/{course_id}")
def get_course_by_id(course_id: int, db: Session = Depends(get_db)):
    return success(course_service.get_course_by_id(db, course_id))


@router.get("/api/v1/admin/courses/search", dependencies=[Depends(require_any_role)])
def search_courses(
    name: str,
    page_number: int = Query(0, alias="page", ge=0),
    size: int = Query(10, ge=1, le=200),
    db: Session = Depends(get_db),
):
    items, total = course_service.search_courses(db, name, page_number, size)
    return success(page(items, total, page_number, size))


@router.post("/api/v1/admin/courses", status_code=status.HTTP_201_CREATED, dependencies=[Depends(require_admin_or_editor)])
def create_course(request: CourseRequest, db: Session = Depends(get_db)):
    return success(course_service.create_course(db, request), "Course created successfully")


@router.put("/api/v1/admin/courses/{course_id}", dependencies=[Depends(require_admin_or_editor)])
def update_course(course_id: int, request: CourseRequest, db: Session = Depends(get_db)):
    return success(course_service.update_course(db, course_id, request), "Course updated successfully")


@router.delete("/api/v1/admin/courses/{course_id}", status_code=status.HTTP_204_NO_CONTENT, dependencies=[Depends(require_admin)])
def delete_course(course_id: int, db: Session = Depends(get_db)):
    course_service.delete_course(db, course_id)
    return Response(status_code=status.HTTP_204_NO_CONTENT)

-------------------------------------------------------------
C:\ws\agent\website-college\backend-python\app\routers\dashboard.py
"""Mirrors controller/DashboardController.java (Module 9 - Admin Dashboard)."""
from fastapi import APIRouter, Depends
from sqlalchemy.orm import Session

from ..database import get_db
from ..responses import success
from ..security import require_any_role
from ..services import dashboard_service

router = APIRouter(prefix="/api/v1/admin/dashboard", tags=["Admin Dashboard"])


@router.get("/summary", dependencies=[Depends(require_any_role)])
def get_summary(db: Session = Depends(get_db)):
    return success(dashboard_service.get_summary(db))

-------------------------------------------------------------
C:\ws\agent\website-college\backend-python\app\routers\faculty.py
"""Mirrors controller/FacultyController.java (Module 5 - Faculty, Module 11 - Faculty Management)."""
from fastapi import APIRouter, Depends, Query, Response, status
from sqlalchemy.orm import Session

from ..database import get_db
from ..responses import page, success
from ..schemas import FacultyRequest
from ..security import require_admin, require_admin_or_editor
from ..services import faculty_service

router = APIRouter(tags=["Faculty"])


@router.get("/api/v1/faculties")
def get_all_faculty(
    page_number: int = Query(0, alias="page", ge=0),
    size: int = Query(10, ge=1, le=200),
    db: Session = Depends(get_db),
):
    items, total = faculty_service.get_all_faculty(db, page_number, size)
    return success(page(items, total, page_number, size))


@router.get("/api/v1/faculties/search")
def search_faculty(
    keyword: str,
    page_number: int = Query(0, alias="page", ge=0),
    size: int = Query(10, ge=1, le=200),
    db: Session = Depends(get_db),
):
    items, total = faculty_service.search_faculty(db, keyword, page_number, size)
    return success(page(items, total, page_number, size))


@router.get("/api/v1/faculties/{faculty_id}")
def get_faculty_by_id(faculty_id: int, db: Session = Depends(get_db)):
    return success(faculty_service.get_faculty_by_id(db, faculty_id))


@router.post("/api/v1/admin/faculties", status_code=status.HTTP_201_CREATED, dependencies=[Depends(require_admin_or_editor)])
def create_faculty(request: FacultyRequest, db: Session = Depends(get_db)):
    return success(faculty_service.create_faculty(db, request), "Faculty created successfully")


@router.put("/api/v1/admin/faculties/{faculty_id}", dependencies=[Depends(require_admin_or_editor)])
def update_faculty(faculty_id: int, request: FacultyRequest, db: Session = Depends(get_db)):
    return success(faculty_service.update_faculty(db, faculty_id, request), "Faculty updated successfully")


@router.delete("/api/v1/admin/faculties/{faculty_id}", status_code=status.HTTP_204_NO_CONTENT, dependencies=[Depends(require_admin)])
def delete_faculty(faculty_id: int, db: Session = Depends(get_db)):
    faculty_service.delete_faculty(db, faculty_id)
    return Response(status_code=status.HTTP_204_NO_CONTENT)

-------------------------------------------------------------
C:\ws\agent\website-college\backend-python\app\routers\fee.py
"""Mirrors controller/FeeController.java (Module 4 - Fee Structure)."""
from fastapi import APIRouter, Depends
from sqlalchemy.orm import Session

from ..database import get_db
from ..responses import success
from ..services import fee_service

router = APIRouter(prefix="/api/v1/fees", tags=["Fees"])


@router.get("")
def get_fees(db: Session = Depends(get_db)):
    return success(fee_service.get_fee_structure(db))

-------------------------------------------------------------
C:\ws\agent\website-college\backend-python\app\routers\home.py
"""Mirrors controller/HomeController.java (Module 1 - Home Page)."""
from fastapi import APIRouter, Depends
from sqlalchemy.orm import Session

from ..database import get_db
from ..responses import success
from ..services import home_service

router = APIRouter(prefix="/api/v1/home", tags=["Home"])


@router.get("")
def get_home(db: Session = Depends(get_db)):
    return success(home_service.get_home_data(db))

-------------------------------------------------------------
C:\ws\agent\website-college\backend-python\app\services\__init__.py
C:\ws\agent\website-college\backend-python\app\services\admission_service.py
"""Mirrors service/impl/AdmissionInquiryServiceImpl.java (Module 7 - Admission
Inquiry, Module 12 - Admission Inquiry Management)."""
from datetime import datetime
from typing import Tuple

from sqlalchemy import or_
from sqlalchemy.orm import Session

from .. import mappers, models
from ..exceptions import InvalidRequestException, ResourceNotFoundException
from ..schemas import AdmissionInquiryRequest, AdmissionInquiryStatusUpdateRequest

VALID_STATUSES = ("NEW", "CONTACTED", "INTERESTED", "VISITED", "ADMITTED", "CLOSED")
CSV_HEADER = (
    "Inquiry Number,Student Name,Mobile,Email,City,State,Course Interest,"
    "Qualification,Status,Created Date\n"
)


def _parse_status(status: str) -> str:
    upper = status.upper()
    if upper not in VALID_STATUSES:
        raise InvalidRequestException(f"Invalid inquiry status: {status}")
    return upper


def _find_or_throw(db: Session, inquiry_id: int) -> models.AdmissionInquiry:
    inquiry = db.get(models.AdmissionInquiry, inquiry_id)
    if inquiry is None:
        raise ResourceNotFoundException.for_entity("Admission inquiry", inquiry_id)
    return inquiry


def submit_inquiry(db: Session, request: AdmissionInquiryRequest) -> dict:
    sequence = db.query(models.AdmissionInquiry).count() + 1
    inquiry_number = f"INQ-{datetime.now().year}-{sequence:06d}"

    inquiry = models.AdmissionInquiry(
        inquiry_number=inquiry_number,
        student_name=request.studentName,
        mobile=request.mobile,
        email=request.email,
        city=request.city,
        state=request.state,
        course_interest=request.courseInterest,
        qualification=request.qualification,
        remarks=request.remarks,
        status="NEW",
    )
    db.add(inquiry)
    db.commit()
    db.refresh(inquiry)
    return mappers.admission_inquiry_to_dto(inquiry)


def get_inquiry_by_id(db: Session, inquiry_id: int) -> dict:
    return mappers.admission_inquiry_to_dto(_find_or_throw(db, inquiry_id))


def get_all_inquiries(db: Session, page: int, size: int) -> Tuple[list, int]:
    query = db.query(models.AdmissionInquiry).order_by(models.AdmissionInquiry.id.asc())
    total = query.count()
    items = query.offset(page * size).limit(size).all()
    return [mappers.admission_inquiry_to_dto(i) for i in items], total


def search_inquiries(db: Session, keyword: str, page: int, size: int) -> Tuple[list, int]:
    like = f"%{keyword}%"
    query = db.query(models.AdmissionInquiry).filter(
        or_(
            models.AdmissionInquiry.student_name.ilike(like),
            models.AdmissionInquiry.mobile.contains(keyword),
            models.AdmissionInquiry.inquiry_number.ilike(like),
        )
    ).order_by(models.AdmissionInquiry.id.asc())
    total = query.count()
    items = query.offset(page * size).limit(size).all()
    return [mappers.admission_inquiry_to_dto(i) for i in items], total


def get_inquiries_by_status(db: Session, status: str, page: int, size: int) -> Tuple[list, int]:
    parsed = _parse_status(status)
    query = db.query(models.AdmissionInquiry).filter(models.AdmissionInquiry.status == parsed).order_by(
        models.AdmissionInquiry.id.asc()
    )
    total = query.count()
    items = query.offset(page * size).limit(size).all()
    return [mappers.admission_inquiry_to_dto(i) for i in items], total


def update_status(db: Session, inquiry_id: int, request: AdmissionInquiryStatusUpdateRequest) -> dict:
    inquiry = _find_or_throw(db, inquiry_id)
    inquiry.status = _parse_status(request.status)
    if request.remarks is not None:
        inquiry.remarks = request.remarks
    db.commit()
    db.refresh(inquiry)
    return mappers.admission_inquiry_to_dto(inquiry)


def _csv_field(value) -> str:
    if value is None:
        return '""'
    escaped = str(value).replace('"', '""')
    return f'"{escaped}"'


def export_all_as_csv(db: Session) -> str:
    lines = [CSV_HEADER]
    inquiries = db.query(models.AdmissionInquiry).order_by(models.AdmissionInquiry.id.asc()).all()
    for inquiry in inquiries:
        created = inquiry.created_date.strftime("%Y-%m-%d %H:%M:%S") if inquiry.created_date else ""
        row = ",".join([
            _csv_field(inquiry.inquiry_number),
            _csv_field(inquiry.student_name),
            _csv_field(inquiry.mobile),
            _csv_field(inquiry.email),
            _csv_field(inquiry.city),
            _csv_field(inquiry.state),
            _csv_field(inquiry.course_interest),
            _csv_field(inquiry.qualification),
            _csv_field(inquiry.status),
            _csv_field(created),
        ])
        lines.append(row + "\n")
    return "".join(lines)

-------------------------------------------------------------
C:\ws\agent\website-college\backend-python\app\services\auth_service.py
"""Mirrors service/impl/AuthServiceImpl.java (Module 8 - Authentication).

Password reset tokens are kept in an in-memory dict for this phase since no
dedicated table exists yet (section 9) - sufficient for a single-instance
deployment, matching the original's ConcurrentHashMap-backed implementation.
"""
import logging
import threading
import uuid
from datetime import datetime, timedelta, timezone
from typing import NamedTuple

from sqlalchemy.orm import Session

from .. import models
from ..config import get_settings
from ..exceptions import AuthenticationFailedException, BadCredentialsException
from ..schemas import ForgotPasswordRequest, LoginRequest, RefreshTokenRequest, ResetPasswordRequest
from ..security import (
    TYPE_REFRESH,
    decode_token,
    generate_access_token,
    generate_refresh_token,
    hash_password,
    verify_password,
)

logger = logging.getLogger("college_portal.auth")

RESET_TOKEN_TTL = timedelta(minutes=30)


class _ResetTokenEntry(NamedTuple):
    username: str
    expires_at: datetime


_reset_tokens: dict[str, _ResetTokenEntry] = {}
_reset_tokens_lock = threading.Lock()


def _build_login_response(username: str, role: str) -> dict:
    settings = get_settings()
    return {
        "accessToken": generate_access_token(username, role),
        "refreshToken": generate_refresh_token(username, role),
        "tokenType": "Bearer",
        "username": username,
        "role": role,
        "expiresInMs": settings.access_token_expiration_ms,
    }


def login(db: Session, request: LoginRequest) -> dict:
    user = db.query(models.User).filter(models.User.username.ilike(request.username)).first()
    if not user or user.status != "ACTIVE" or not verify_password(request.password, user.password):
        raise BadCredentialsException()
    return _build_login_response(user.username, user.role)


def refresh_token(db: Session, request: RefreshTokenRequest) -> dict:
    claims = decode_token(request.refreshToken)
    if not claims or claims.get("type") != TYPE_REFRESH:
        raise AuthenticationFailedException("Refresh token is invalid or expired")

    username = claims.get("sub")
    user = db.query(models.User).filter(models.User.username.ilike(username)).first()
    if not user:
        raise AuthenticationFailedException("User no longer exists")
    return _build_login_response(user.username, user.role)


def forgot_password(db: Session, request: ForgotPasswordRequest) -> None:
    settings = get_settings()
    user = db.query(models.User).filter(models.User.username.ilike(request.usernameOrEmail)).first()
    if user:
        token = str(uuid.uuid4())
        with _reset_tokens_lock:
            _reset_tokens[token] = _ResetTokenEntry(
                username=user.username,
                expires_at=datetime.now(timezone.utc) + RESET_TOKEN_TTL,
            )
        if settings.feature_email_enabled:
            logger.info("Password reset email sent to user '%s' (token omitted from logs)", user.username)
        else:
            logger.info(
                "Email sending disabled (feature.email-enabled=false); reset token for '%s': %s",
                user.username, token,
            )
    # Always return success regardless of whether the user exists, to avoid
    # leaking which usernames/emails are registered.


def reset_password(db: Session, request: ResetPasswordRequest) -> None:
    with _reset_tokens_lock:
        entry = _reset_tokens.get(request.token)
        if entry is None or entry.expires_at < datetime.now(timezone.utc):
            _reset_tokens.pop(request.token, None)
            raise AuthenticationFailedException("Reset token is invalid or expired")
        del _reset_tokens[request.token]

    user = db.query(models.User).filter(models.User.username.ilike(entry.username)).first()
    if not user:
        raise AuthenticationFailedException("User no longer exists")

    user.password = hash_password(request.newPassword)
    db.commit()

-------------------------------------------------------------
C:\ws\agent\website-college\backend-python\app\services\college_service.py
"""Mirrors service/impl/CollegeInfoServiceImpl.java (Module 2 - About College)."""
from sqlalchemy.orm import Session

from .. import mappers, models
from ..exceptions import ResourceNotFoundException


def get_college_info(db: Session) -> dict:
    info = db.query(models.CollegeInfo).first()
    if info is None:
        raise ResourceNotFoundException("College information has not been configured yet")
    return mappers.college_info_to_dto(info)

-------------------------------------------------------------
C:\ws\agent\website-college\backend-python\app\services\contact_service.py
"""Mirrors service/impl/ContactUsServiceImpl.java (Module 6 - Contact Us)."""
from typing import Tuple

from sqlalchemy.orm import Session

from .. import mappers, models
from ..exceptions import ResourceNotFoundException
from ..schemas import ContactUsRequest


def submit_contact(db: Session, request: ContactUsRequest) -> dict:
    contact = models.ContactUs(
        name=request.name,
        email=request.email,
        mobile=request.mobile,
        subject=request.subject,
        message=request.message,
    )
    db.add(contact)
    db.commit()
    db.refresh(contact)
    return mappers.contact_us_to_dto(contact)


def get_all_contacts(db: Session, page: int, size: int) -> Tuple[list, int]:
    query = db.query(models.ContactUs).order_by(models.ContactUs.id.asc())
    total = query.count()
    items = query.offset(page * size).limit(size).all()
    return [mappers.contact_us_to_dto(c) for c in items], total


def get_contact_by_id(db: Session, contact_id: int) -> dict:
    contact = db.get(models.ContactUs, contact_id)
    if contact is None:
        raise ResourceNotFoundException.for_entity("Contact", contact_id)
    return mappers.contact_us_to_dto(contact)

-------------------------------------------------------------
C:\ws\agent\website-college\backend-python\app\services\course_service.py
"""Mirrors service/impl/CourseServiceImpl.java (Module 3 - Courses, Module 10 - Course Management)."""
from typing import Tuple

from sqlalchemy.orm import Session

from .. import mappers, models
from ..exceptions import DuplicateResourceException, InvalidRequestException, ResourceNotFoundException
from ..schemas import CourseRequest

VALID_STATUSES = ("ACTIVE", "INACTIVE")


def _validate_status(status: str) -> str:
    upper = status.upper()
    if upper not in VALID_STATUSES:
        raise InvalidRequestException(f"No enum constant CourseStatus.{upper}")
    return upper


def _find_or_throw(db: Session, course_id: int) -> models.Course:
    course = db.get(models.Course, course_id)
    if course is None:
        raise ResourceNotFoundException.for_entity("Course", course_id)
    return course


def get_all_courses(db: Session, page: int, size: int) -> Tuple[list, int]:
    query = db.query(models.Course).order_by(models.Course.id.asc())
    total = query.count()
    items = query.offset(page * size).limit(size).all()
    return [mappers.course_to_dto(c) for c in items], total


def search_courses(db: Session, name: str, page: int, size: int) -> Tuple[list, int]:
    query = db.query(models.Course).filter(models.Course.course_name.ilike(f"%{name}%")).order_by(
        models.Course.id.asc()
    )
    total = query.count()
    items = query.offset(page * size).limit(size).all()
    return [mappers.course_to_dto(c) for c in items], total


def get_course_by_id(db: Session, course_id: int) -> dict:
    return mappers.course_to_dto(_find_or_throw(db, course_id))


def create_course(db: Session, request: CourseRequest) -> dict:
    status = _validate_status(request.status)
    existing = db.query(models.Course).filter(models.Course.course_code.ilike(request.courseCode)).first()
    if existing:
        raise DuplicateResourceException(f"Course with code {request.courseCode} already exists")

    course = models.Course(
        course_code=request.courseCode,
        course_name=request.courseName,
        duration=request.duration,
        eligibility=request.eligibility,
        fees=request.fees,
        description=request.description,
        intake_capacity=request.intakeCapacity,
        career_opportunities=request.careerOpportunities,
        status=status,
    )
    db.add(course)
    db.commit()
    db.refresh(course)
    return mappers.course_to_dto(course)


def update_course(db: Session, course_id: int, request: CourseRequest) -> dict:
    course = _find_or_throw(db, course_id)
    status = _validate_status(request.status)

    if course.course_code.lower() != request.courseCode.lower():
        existing = db.query(models.Course).filter(models.Course.course_code.ilike(request.courseCode)).first()
        if existing:
            raise DuplicateResourceException(f"Course with code {request.courseCode} already exists")

    course.course_code = request.courseCode
    course.course_name = request.courseName
    course.duration = request.duration
    course.eligibility = request.eligibility
    course.fees = request.fees
    course.description = request.description
    course.intake_capacity = request.intakeCapacity
    course.career_opportunities = request.careerOpportunities
    course.status = status

    db.commit()
    db.refresh(course)
    return mappers.course_to_dto(course)


def delete_course(db: Session, course_id: int) -> None:
    course = _find_or_throw(db, course_id)
    db.delete(course)
    db.commit()

-------------------------------------------------------------
C:\ws\agent\website-college\backend-python\app\services\dashboard_service.py
"""Mirrors service/impl/DashboardServiceImpl.java (Module 9 - Admin Dashboard).

Aggregation is performed in-memory (as in the original, which favored
DB-agnostic Java streams over SQL-specific functions ahead of a planned
Cloud SQL migration - instructions section 17).
"""
from collections import Counter
from typing import Callable, List

from sqlalchemy.orm import Session

from .. import models


def _group_and_count(inquiries: List[models.AdmissionInquiry], classifier: Callable[[models.AdmissionInquiry], str]) -> list:
    counts = Counter(classifier(i) for i in inquiries)
    return [{"label": label, "value": value} for label, value in sorted(counts.items())]


def get_summary(db: Session) -> dict:
    inquiries = db.query(models.AdmissionInquiry).all()

    return {
        "totalCourses": db.query(models.Course).count(),
        "totalFaculty": db.query(models.Faculty).count(),
        "totalAdmissionInquiries": len(inquiries),
        "totalContacts": db.query(models.ContactUs).count(),
        "monthlyInquiries": _group_and_count(inquiries, lambda i: i.created_date.strftime("%Y-%m")),
        "courseWiseInquiries": _group_and_count(inquiries, lambda i: i.course_interest),
        "dailyRequests": _group_and_count(inquiries, lambda i: i.created_date.strftime("%Y-%m-%d")),
    }

-------------------------------------------------------------
C:\ws\agent\website-college\backend-python\app\services\faculty_service.py
"""Mirrors service/impl/FacultyServiceImpl.java (Module 5 - Faculty, Module 11 - Faculty Management)."""
from typing import Tuple

from sqlalchemy import or_
from sqlalchemy.orm import Session

from .. import mappers, models
from ..exceptions import ResourceNotFoundException
from ..schemas import FacultyRequest


def _find_or_throw(db: Session, faculty_id: int) -> models.Faculty:
    faculty = db.get(models.Faculty, faculty_id)
    if faculty is None:
        raise ResourceNotFoundException.for_entity("Faculty", faculty_id)
    return faculty


def get_all_faculty(db: Session, page: int, size: int) -> Tuple[list, int]:
    query = db.query(models.Faculty).order_by(models.Faculty.id.asc())
    total = query.count()
    items = query.offset(page * size).limit(size).all()
    return [mappers.faculty_to_dto(f) for f in items], total


def search_faculty(db: Session, keyword: str, page: int, size: int) -> Tuple[list, int]:
    like = f"%{keyword}%"
    query = db.query(models.Faculty).filter(
        or_(models.Faculty.name.ilike(like), models.Faculty.department.ilike(like))
    ).order_by(models.Faculty.id.asc())
    total = query.count()
    items = query.offset(page * size).limit(size).all()
    return [mappers.faculty_to_dto(f) for f in items], total


def get_faculty_by_id(db: Session, faculty_id: int) -> dict:
    return mappers.faculty_to_dto(_find_or_throw(db, faculty_id))


def create_faculty(db: Session, request: FacultyRequest) -> dict:
    faculty = models.Faculty(
        name=request.name,
        department=request.department,
        designation=request.designation,
        qualification=request.qualification,
        experience=request.experience,
        specialization=request.specialization,
        photo_url=request.photoUrl,
    )
    db.add(faculty)
    db.commit()
    db.refresh(faculty)
    return mappers.faculty_to_dto(faculty)


def update_faculty(db: Session, faculty_id: int, request: FacultyRequest) -> dict:
    faculty = _find_or_throw(db, faculty_id)
    faculty.name = request.name
    faculty.department = request.department
    faculty.designation = request.designation
    faculty.qualification = request.qualification
    faculty.experience = request.experience
    faculty.specialization = request.specialization
    faculty.photo_url = request.photoUrl
    db.commit()
    db.refresh(faculty)
    return mappers.faculty_to_dto(faculty)


def delete_faculty(db: Session, faculty_id: int) -> None:
    faculty = _find_or_throw(db, faculty_id)
    db.delete(faculty)
    db.commit()

-------------------------------------------------------------
C:\ws\agent\website-college\backend-python\app\services\fee_service.py
"""Mirrors service/impl/FeeServiceImpl.java (Module 4 - Fee Structure)."""
from sqlalchemy.orm import Session

from .. import models
from ..config import get_settings


def get_fee_structure(db: Session) -> dict:
    settings = get_settings()
    courses = db.query(models.Course).order_by(models.Course.id.asc()).all()
    course_fees = [
        {
            "courseCode": c.course_code,
            "courseName": c.course_name,
            "semesterFees": float(c.fees) if c.fees is not None else None,
        }
        for c in courses
    ]

    return {
        "courseFees": course_fees,
        "admissionFees": float(settings.admission_fees),
        "hostelFees": float(settings.hostel_fees),
        "transportFees": float(settings.transport_fees),
        "scholarshipInformation": settings.scholarship_information,
    }

-------------------------------------------------------------
C:\ws\agent\website-college\backend-python\app\services\home_service.py
"""Mirrors service/impl/HomeServiceImpl.java (Module 1 - Home Page).

News/events/highlights are static sample content, since no dedicated CMS
tables exist yet in the database design (section 9).
"""
from datetime import datetime

from sqlalchemy.orm import Session

from .. import models


def get_home_data(db: Session) -> dict:
    info = db.query(models.CollegeInfo).first()
    name = info.name if info else "College Digital Foundation Portal"
    vision = info.vision if info else None
    mission = info.mission if info else None

    return {
        "collegeName": name,
        "logoUrl": "/assets/images/college-logo.png",
        "vision": vision,
        "mission": mission,
        "principalMessage": (
            f"Welcome to {name}. We are committed to nurturing future leaders "
            "through quality education, strong values, and a supportive learning environment."
        ),
        "latestNews": [
            "Admissions open for the new academic year",
            "Annual sports meet scheduled next month",
            "New computer lab inaugurated",
        ],
        "upcomingEvents": [
            "Orientation Day",
            "Guest Lecture Series",
            "Cultural Fest",
        ],
        "collegeHighlights": [
            "NAAC Accredited",
            "Modern Infrastructure",
            "Experienced Faculty",
            "Industry Partnerships",
        ],
        "placementHighlights": [
            "95% Placement Rate",
            "200+ Recruiting Companies",
            "Highest Package: 18 LPA",
        ],
        "admissionBannerText": f"Admissions Open {datetime.now().year} - Apply Now!",
    }

-------------------------------------------------------------
C:\ws\agent\website-college\backend-python\app\__init__.py
C:\ws\agent\website-college\backend-python\app\config.py
"""Application configuration.

Mirrors application.yml / application-local.yml / application-gcp.yml and the
`app.jwt.*`, `app.cors.*`, `app.fees.*`, `feature.*` property groups from the
original Spring Boot backend (instructions sections 11A, 12).
"""
import os
from decimal import Decimal
from functools import lru_cache
from typing import List


class Settings:
    def __init__(self) -> None:
        # Equivalent of SPRING_PROFILES_ACTIVE (local | gcp).
        self.profile = os.getenv("APP_PROFILE", os.getenv("SPRING_PROFILES_ACTIVE", "local")).lower()
        self.port = int(os.getenv("PORT", "8080"))

        self.jwt_secret = os.getenv(
            "JWT_SECRET", "local-dev-only-change-me-must-be-32-bytes-minimum-secret"
        )
        self.access_token_expiration_ms = int(os.getenv("ACCESS_TOKEN_EXPIRATION_MS", "3600000"))
        self.refresh_token_expiration_ms = int(os.getenv("REFRESH_TOKEN_EXPIRATION_MS", "604800000"))

        origins = os.getenv("CORS_ALLOWED_ORIGINS", "http://localhost:4200")
        self.cors_allowed_origins: List[str] = [o.strip() for o in origins.split(",") if o.strip()]

        is_gcp = self.profile == "gcp"
        self.feature_cloud_run_enabled = is_gcp
        self.feature_analytics_enabled = is_gcp
        self.feature_email_enabled = is_gcp

        # Static fee configuration (no dedicated FEE table exists - section 9).
        self.admission_fees = Decimal("15000.00")
        self.hostel_fees = Decimal("60000.00")
        self.transport_fees = Decimal("20000.00")
        self.scholarship_information = (
            "Merit-based and need-based scholarships up to 50% of tuition fees "
            "are available for eligible students."
        )


@lru_cache
def get_settings() -> Settings:
    return Settings()

-------------------------------------------------------------
C:\ws\agent\website-college\backend-python\app\constants.py
"""Role name constants, mirroring com.college.portal.constants.Roles."""


class Roles:
    ADMIN = "ROLE_ADMIN"
    EDITOR = "ROLE_EDITOR"
    VIEWER = "ROLE_VIEWER"

    ALL = (ADMIN, EDITOR, VIEWER)
    ADMIN_EDITOR = (ADMIN, EDITOR)

-------------------------------------------------------------
C:\ws\agent\website-college\backend-python\app\database.py
"""SQLAlchemy engine/session setup.

Defaults to a single shared in-memory SQLite database (via StaticPool) to
mirror the original `jdbc:h2:mem:collegedb;DB_CLOSE_DELAY=-1` local H2
configuration: data lives only for the lifetime of the running process.

Set the `DATABASE_URL` environment variable (e.g.
`postgresql+psycopg2://user:password@host:5432/collegedb`) to instead run
against a real PostgreSQL database (Cloud SQL or local) - no other code
changes are required, since no SQLite/H2-specific SQL is used anywhere.
"""
import os

from sqlalchemy import create_engine
from sqlalchemy.orm import DeclarativeBase, sessionmaker
from sqlalchemy.pool import StaticPool

DATABASE_URL = os.getenv("DATABASE_URL", "sqlite://")

if DATABASE_URL.startswith("sqlite"):
    engine = create_engine(
        DATABASE_URL,
        connect_args={"check_same_thread": False},
        poolclass=StaticPool,
    )
else:
    engine = create_engine(DATABASE_URL, pool_pre_ping=True)

SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)


class Base(DeclarativeBase):
    pass


def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

-------------------------------------------------------------
C:\ws\agent\website-college\backend-python\app\exceptions.py
"""Custom exceptions, mirroring the exception/ package.

Mapped to the standard error envelope (instructions section 4B) by the
exception handlers registered in main.py (mirrors GlobalExceptionHandler,
RestAuthenticationEntryPoint and RestAccessDeniedHandler).
"""


class ResourceNotFoundException(Exception):
    """Mapped to HTTP 404 RESOURCE_NOT_FOUND."""

    def __init__(self, message: str):
        self.message = message
        super().__init__(message)

    @classmethod
    def for_entity(cls, entity_name: str, entity_id) -> "ResourceNotFoundException":
        return cls(f"{entity_name} with id {entity_id} not found")


class DuplicateResourceException(Exception):
    """Mapped to HTTP 409 DUPLICATE_RESOURCE."""

    def __init__(self, message: str):
        self.message = message
        super().__init__(message)


class AuthenticationFailedException(Exception):
    """Mapped to HTTP 401 AUTHENTICATION_FAILED."""

    def __init__(self, message: str):
        self.message = message
        super().__init__(message)


class BadCredentialsException(Exception):
    """Raised on login failure. Mapped to HTTP 401 INVALID_CREDENTIALS."""


class UnauthorizedException(Exception):
    """No/invalid JWT on a protected endpoint. Mapped to HTTP 401 UNAUTHORIZED."""


class AccessDeniedException(Exception):
    """Authenticated but insufficient role. Mapped to HTTP 403 ACCESS_DENIED."""


class InvalidRequestException(Exception):
    """Mirrors IllegalArgumentException handling. Mapped to HTTP 400 INVALID_REQUEST."""

    def __init__(self, message: str):
        self.message = message
        super().__init__(message)

-------------------------------------------------------------
C:\ws\agent\website-college\backend-python\app\main.py
"""FastAPI application entrypoint.

Mirrors CollegePortalApplication.java + SecurityConfig.java (CORS, exception
mapping) + config/DataInitializer.java (startup seeding). Wires together all
routers so the exposed REST contract is identical to the original Spring Boot
backend (instructions section 4B) and requires no Angular frontend changes.
"""
import logging

from fastapi import FastAPI, Request
from fastapi.exceptions import RequestValidationError
from fastapi.middleware.cors import CORSMiddleware
from fastapi.responses import JSONResponse
from starlette.exceptions import HTTPException as StarletteHTTPException

from .config import get_settings
from .exceptions import (
    AccessDeniedException,
    AuthenticationFailedException,
    BadCredentialsException,
    DuplicateResourceException,
    InvalidRequestException,
    ResourceNotFoundException,
    UnauthorizedException,
)
from .responses import error
from .routers import admission, auth, college, contact, course, dashboard, fee, faculty, home
from .seed import seed_all

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("college_portal")

settings = get_settings()

app = FastAPI(
    title="College Digital Foundation Portal API",
    version="1.0.0",
    docs_url="/swagger-ui.html",
    openapi_url="/v3/api-docs",
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.cors_allowed_origins,
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS"],
    allow_headers=["Authorization", "Content-Type", "Accept"],
)


@app.on_event("startup")
def on_startup() -> None:
    seed_all()
    logger.info("Database schema created and sample data seeded (profile=%s)", settings.profile)


@app.get("/actuator/health", tags=["Health"])
def health():
    return {"status": "UP"}


# --- Exception handlers, mirroring GlobalExceptionHandler / RestAuthenticationEntryPoint / RestAccessDeniedHandler ---

@app.exception_handler(ResourceNotFoundException)
async def handle_not_found(request: Request, exc: ResourceNotFoundException):
    return JSONResponse(status_code=404, content=error("RESOURCE_NOT_FOUND", exc.message))


@app.exception_handler(DuplicateResourceException)
async def handle_duplicate(request: Request, exc: DuplicateResourceException):
    return JSONResponse(status_code=409, content=error("DUPLICATE_RESOURCE", exc.message))


@app.exception_handler(AuthenticationFailedException)
async def handle_auth_failed(request: Request, exc: AuthenticationFailedException):
    return JSONResponse(status_code=401, content=error("AUTHENTICATION_FAILED", exc.message))


@app.exception_handler(BadCredentialsException)
async def handle_bad_credentials(request: Request, exc: BadCredentialsException):
    return JSONResponse(status_code=401, content=error("INVALID_CREDENTIALS", "Invalid username or password"))


@app.exception_handler(UnauthorizedException)
async def handle_unauthorized(request: Request, exc: UnauthorizedException):
    return JSONResponse(
        status_code=401,
        content=error("UNAUTHORIZED", "Authentication is required to access this resource"),
    )


@app.exception_handler(AccessDeniedException)
async def handle_access_denied(request: Request, exc: AccessDeniedException):
    return JSONResponse(
        status_code=403,
        content=error("ACCESS_DENIED", "You do not have permission to perform this action"),
    )


@app.exception_handler(InvalidRequestException)
async def handle_invalid_request(request: Request, exc: InvalidRequestException):
    return JSONResponse(status_code=400, content=error("INVALID_REQUEST", exc.message))


@app.exception_handler(RequestValidationError)
async def handle_validation_error(request: Request, exc: RequestValidationError):
    details = []
    for err in exc.errors():
        field = err["loc"][-1] if err["loc"] else "request"
        details.append(f"{field}: {err['msg']}")
    return JSONResponse(status_code=400, content=error("VALIDATION_FAILED", "Request validation failed", details))


@app.exception_handler(StarletteHTTPException)
async def handle_http_exception(request: Request, exc: StarletteHTTPException):
    if exc.status_code == 404:
        return JSONResponse(status_code=404, content=error("RESOURCE_NOT_FOUND", "The requested resource was not found"))
    if exc.status_code == 405:
        return JSONResponse(status_code=405, content=error("METHOD_NOT_ALLOWED", str(exc.detail)))
    return JSONResponse(status_code=exc.status_code, content=error("REQUEST_FAILED", str(exc.detail)))


@app.exception_handler(Exception)
async def handle_generic_exception(request: Request, exc: Exception):
    logger.exception("Unhandled exception")
    return JSONResponse(
        status_code=500,
        content=error("INTERNAL_SERVER_ERROR", "An unexpected error occurred. Please try again later."),
    )


# --- Routers, mirroring the controller/ package ---
app.include_router(home.router)
app.include_router(college.router)
app.include_router(fee.router)
app.include_router(course.router)
app.include_router(faculty.router)
app.include_router(contact.router)
app.include_router(admission.router)
app.include_router(dashboard.router)
app.include_router(auth.router)

-------------------------------------------------------------
C:\ws\agent\website-college\backend-python\app\mappers.py
"""Entity -> response DTO (dict) mappers, mirroring the mapper/ package.

Plain dicts (rather than Pydantic response models) are used so numeric/date
serialization can be controlled precisely to match the original Jackson
output that the Angular frontend already expects (numbers, not strings, for
money fields; ISO-8601 strings for timestamps).
"""
from __future__ import annotations

from typing import Optional

from . import models


def _money(value) -> Optional[float]:
    return float(value) if value is not None else None


def _iso(dt) -> Optional[str]:
    return dt.isoformat() if dt is not None else None


def college_info_to_dto(entity: models.CollegeInfo) -> dict:
    return {
        "id": entity.id,
        "name": entity.name,
        "vision": entity.vision,
        "mission": entity.mission,
        "aboutUs": entity.about_us,
        "address": entity.address,
        "phone": entity.phone,
        "email": entity.email,
        "website": entity.website,
    }


def course_to_dto(entity: models.Course) -> dict:
    return {
        "id": entity.id,
        "courseCode": entity.course_code,
        "courseName": entity.course_name,
        "duration": entity.duration,
        "eligibility": entity.eligibility,
        "fees": _money(entity.fees),
        "description": entity.description,
        "intakeCapacity": entity.intake_capacity,
        "careerOpportunities": entity.career_opportunities,
        "status": entity.status,
    }


def faculty_to_dto(entity: models.Faculty) -> dict:
    return {
        "id": entity.id,
        "name": entity.name,
        "department": entity.department,
        "designation": entity.designation,
        "qualification": entity.qualification,
        "experience": entity.experience,
        "specialization": entity.specialization,
        "photoUrl": entity.photo_url,
    }


def contact_us_to_dto(entity: models.ContactUs) -> dict:
    return {
        "id": entity.id,
        "name": entity.name,
        "email": entity.email,
        "mobile": entity.mobile,
        "subject": entity.subject,
        "message": entity.message,
        "createdDate": _iso(entity.created_date),
    }


def admission_inquiry_to_dto(entity: models.AdmissionInquiry) -> dict:
    return {
        "id": entity.id,
        "inquiryNumber": entity.inquiry_number,
        "studentName": entity.student_name,
        "mobile": entity.mobile,
        "email": entity.email,
        "city": entity.city,
        "state": entity.state,
        "courseInterest": entity.course_interest,
        "qualification": entity.qualification,
        "remarks": entity.remarks,
        "status": entity.status,
        "createdDate": _iso(entity.created_date),
    }

-------------------------------------------------------------
C:\ws\agent\website-college\backend-python\app\models.py
"""SQLAlchemy ORM models, mirroring the entity/ package and schema.sql (section 9).

Enum-like columns (course/inquiry/user status, user role) are stored as plain
strings and validated in the service layer, mirroring the Java
@Enumerated(EnumType.STRING) + IllegalArgumentException-on-invalid-value behavior.
"""
from datetime import datetime, timezone

from sqlalchemy import Column, DateTime, Integer, Numeric, String, Text

from .database import Base


def _utcnow() -> datetime:
    return datetime.now(timezone.utc).replace(tzinfo=None)


class CollegeInfo(Base):
    __tablename__ = "college_info"

    id = Column(Integer, primary_key=True, autoincrement=True)
    name = Column(String(255), nullable=False)
    vision = Column(String(2000))
    mission = Column(String(2000))
    about_us = Column(Text)
    address = Column(String(500))
    phone = Column(String(50))
    email = Column(String(255))
    website = Column(String(255))


class Course(Base):
    __tablename__ = "course"

    id = Column(Integer, primary_key=True, autoincrement=True)
    course_code = Column(String(50), nullable=False, unique=True)
    course_name = Column(String(255), nullable=False)
    duration = Column(String(100))
    eligibility = Column(String(500))
    fees = Column(Numeric(12, 2))
    description = Column(Text)
    intake_capacity = Column(Integer)
    career_opportunities = Column(String(1000))
    status = Column(String(20), nullable=False, default="ACTIVE")


class Faculty(Base):
    __tablename__ = "faculty"

    id = Column(Integer, primary_key=True, autoincrement=True)
    name = Column(String(255), nullable=False)
    department = Column(String(255))
    designation = Column(String(255))
    qualification = Column(String(255))
    experience = Column(Integer)
    specialization = Column(String(500))
    photo_url = Column(String(500))


class ContactUs(Base):
    __tablename__ = "contact_us"

    id = Column(Integer, primary_key=True, autoincrement=True)
    name = Column(String(255), nullable=False)
    email = Column(String(255), nullable=False)
    mobile = Column(String(20))
    subject = Column(String(255))
    message = Column(Text)
    created_date = Column(DateTime, nullable=False, default=_utcnow)


class AdmissionInquiry(Base):
    __tablename__ = "admission_inquiry"

    id = Column(Integer, primary_key=True, autoincrement=True)
    inquiry_number = Column(String(50), nullable=False, unique=True)
    student_name = Column(String(255), nullable=False)
    mobile = Column(String(20), nullable=False)
    email = Column(String(255))
    city = Column(String(100))
    state = Column(String(100))
    course_interest = Column(String(255))
    qualification = Column(String(255))
    remarks = Column(String(2000))
    status = Column(String(50), nullable=False, default="NEW")
    created_date = Column(DateTime, nullable=False, default=_utcnow)


class User(Base):
    __tablename__ = "users"

    id = Column(Integer, primary_key=True, autoincrement=True)
    username = Column(String(255), nullable=False, unique=True)
    password = Column(String(500), nullable=False)
    role = Column(String(50), nullable=False)
    status = Column(String(20), nullable=False, default="ACTIVE")

-------------------------------------------------------------
C:\ws\agent\website-college\backend-python\app\responses.py
"""Standard API response envelope builders, mirroring dto/common/
(ApiResponse, ErrorResponse, PageResponse) per instructions section 4B.
"""
from __future__ import annotations

from datetime import datetime, timezone
from typing import Any, Iterable, Optional


def _timestamp() -> str:
    return datetime.now(timezone.utc).isoformat()


def success(data: Any = None, message: str = "Operation successful") -> dict:
    return {
        "success": True,
        "data": data,
        "message": message,
        "timestamp": _timestamp(),
    }


def error(error_code: str, message: str, details: Optional[Iterable[str]] = None) -> dict:
    return {
        "success": False,
        "errorCode": error_code,
        "message": message,
        "details": list(details) if details else [],
        "timestamp": _timestamp(),
    }


def page(content: list, total_elements: int, page_number: int, size: int) -> dict:
    total_pages = (total_elements + size - 1) // size if size > 0 else 0
    return {
        "content": content,
        "totalElements": total_elements,
        "totalPages": total_pages,
        "page": page_number,
        "size": size,
    }

-------------------------------------------------------------
C:\ws\agent\website-college\backend-python\app\schemas.py
"""Request DTOs (Pydantic models), mirroring the dto/ and dto/auth/ packages.

Validation constraints mirror the Jakarta Bean Validation annotations used in
the original DTOs exactly (message text included), so 400 VALIDATION_FAILED
responses stay compatible with the Angular error interceptor.
"""
from __future__ import annotations

import re
from typing import Annotated, Optional

from pydantic import BaseModel, EmailStr, Field, field_validator

MOBILE_PATTERN = re.compile(r"^[0-9+\-() ]{7,20}$")


def _not_blank(value: str, message: str = "must not be blank") -> str:
    if value is None or value.strip() == "":
        raise ValueError(message)
    return value


def _valid_mobile(value: str) -> str:
    if not MOBILE_PATTERN.match(value):
        raise ValueError("Mobile number is invalid")
    return value


class AdmissionInquiryRequest(BaseModel):
    studentName: Annotated[str, Field(max_length=255)]
    mobile: Annotated[str, Field(max_length=20)]
    email: Optional[EmailStr] = None
    city: Optional[Annotated[str, Field(max_length=100)]] = None
    state: Optional[Annotated[str, Field(max_length=100)]] = None
    courseInterest: Annotated[str, Field(max_length=255)]
    qualification: Optional[Annotated[str, Field(max_length=255)]] = None
    remarks: Optional[Annotated[str, Field(max_length=2000)]] = None

    @field_validator("email", mode="before")
    @classmethod
    def _blank_email_to_none(cls, v):
        # Mirrors Jakarta's @Email, which treats null/blank as valid (only
        # @NotBlank rejects blank) - the Angular form allows submitting "".
        if v is None or (isinstance(v, str) and v.strip() == ""):
            return None
        return v

    @field_validator("studentName")
    @classmethod
    def _student_name_not_blank(cls, v: str) -> str:
        return _not_blank(v, "Student name is required")

    @field_validator("mobile")
    @classmethod
    def _mobile_valid(cls, v: str) -> str:
        _not_blank(v, "Mobile is required")
        return _valid_mobile(v)

    @field_validator("courseInterest")
    @classmethod
    def _course_interest_not_blank(cls, v: str) -> str:
        return _not_blank(v, "Course interested is required")


class AdmissionInquiryStatusUpdateRequest(BaseModel):
    status: str
    remarks: Optional[Annotated[str, Field(max_length=2000)]] = None

    @field_validator("status")
    @classmethod
    def _status_not_blank(cls, v: str) -> str:
        return _not_blank(v, "Status is required")


class ContactUsRequest(BaseModel):
    name: Annotated[str, Field(max_length=255)]
    mobile: Annotated[str, Field(max_length=20)]
    email: EmailStr
    subject: Annotated[str, Field(max_length=255)]
    message: Annotated[str, Field(max_length=4000)]

    @field_validator("name")
    @classmethod
    def _name_not_blank(cls, v: str) -> str:
        return _not_blank(v, "Name is required")

    @field_validator("mobile")
    @classmethod
    def _mobile_valid(cls, v: str) -> str:
        _not_blank(v, "Mobile is required")
        return _valid_mobile(v)

    @field_validator("subject")
    @classmethod
    def _subject_not_blank(cls, v: str) -> str:
        return _not_blank(v, "Subject is required")

    @field_validator("message")
    @classmethod
    def _message_not_blank(cls, v: str) -> str:
        return _not_blank(v, "Message is required")


class CourseRequest(BaseModel):
    courseCode: Annotated[str, Field(max_length=50)]
    courseName: Annotated[str, Field(max_length=255)]
    duration: Optional[Annotated[str, Field(max_length=100)]] = None
    eligibility: Optional[Annotated[str, Field(max_length=500)]] = None
    fees: Annotated[float, Field(ge=0.0)]
    description: Optional[str] = None
    intakeCapacity: Optional[Annotated[int, Field(gt=0)]] = None
    careerOpportunities: Optional[Annotated[str, Field(max_length=1000)]] = None
    status: str

    @field_validator("courseCode")
    @classmethod
    def _course_code_not_blank(cls, v: str) -> str:
        return _not_blank(v, "Course code is required")

    @field_validator("courseName")
    @classmethod
    def _course_name_not_blank(cls, v: str) -> str:
        return _not_blank(v, "Course name is required")

    @field_validator("status")
    @classmethod
    def _status_not_blank(cls, v: str) -> str:
        return _not_blank(v, "Status is required")


class FacultyRequest(BaseModel):
    name: Annotated[str, Field(max_length=255)]
    department: Annotated[str, Field(max_length=255)]
    designation: Optional[Annotated[str, Field(max_length=255)]] = None
    qualification: Optional[Annotated[str, Field(max_length=255)]] = None
    experience: Optional[Annotated[int, Field(ge=0)]] = None
    specialization: Optional[Annotated[str, Field(max_length=500)]] = None
    photoUrl: Optional[Annotated[str, Field(max_length=500)]] = None

    @field_validator("name")
    @classmethod
    def _name_not_blank(cls, v: str) -> str:
        return _not_blank(v, "Name is required")

    @field_validator("department")
    @classmethod
    def _department_not_blank(cls, v: str) -> str:
        return _not_blank(v, "Department is required")


class LoginRequest(BaseModel):
    username: str
    password: str

    @field_validator("username")
    @classmethod
    def _username_not_blank(cls, v: str) -> str:
        return _not_blank(v, "Username is required")

    @field_validator("password")
    @classmethod
    def _password_not_blank(cls, v: str) -> str:
        return _not_blank(v, "Password is required")


class RefreshTokenRequest(BaseModel):
    refreshToken: str

    @field_validator("refreshToken")
    @classmethod
    def _refresh_token_not_blank(cls, v: str) -> str:
        return _not_blank(v, "Refresh token is required")


class ForgotPasswordRequest(BaseModel):
    usernameOrEmail: str

    @field_validator("usernameOrEmail")
    @classmethod
    def _username_or_email_not_blank(cls, v: str) -> str:
        return _not_blank(v, "Email/username is required")


class ResetPasswordRequest(BaseModel):
    token: str
    newPassword: Annotated[str, Field(min_length=6, max_length=100)]

    @field_validator("token")
    @classmethod
    def _token_not_blank(cls, v: str) -> str:
        return _not_blank(v, "Reset token is required")

-------------------------------------------------------------
C:\ws\agent\website-college\backend-python\app\security.py
"""JWT issuing/validation and password hashing.

Mirrors the security/ package (JwtTokenProvider, JwtAuthenticationFilter,
CustomUserDetails, CustomUserDetailsService) plus the RBAC dependency
equivalents of @PreAuthorize("hasAnyRole(...)")/hasRole(...) used by
SecurityConfig (instructions sections 8, 11A).
"""
from __future__ import annotations

import time
from typing import Optional

import bcrypt
import jwt
from fastapi import Depends, Header
from sqlalchemy.orm import Session

from .config import get_settings
from .constants import Roles
from .database import get_db
from .exceptions import AccessDeniedException, UnauthorizedException
from .models import User

ALGORITHM = "HS256"
TYPE_ACCESS = "access"
TYPE_REFRESH = "refresh"
BCRYPT_ROUNDS = 10


def hash_password(raw_password: str) -> str:
    return bcrypt.hashpw(raw_password.encode("utf-8"), bcrypt.gensalt(rounds=BCRYPT_ROUNDS)).decode("utf-8")


def verify_password(raw_password: str, hashed_password: str) -> bool:
    try:
        return bcrypt.checkpw(raw_password.encode("utf-8"), hashed_password.encode("utf-8"))
    except ValueError:
        return False


def _build_token(username: str, role: str, token_type: str, expiration_ms: int) -> str:
    now = int(time.time())
    payload = {
        "sub": username,
        "role": role,
        "type": token_type,
        "iat": now,
        "exp": now + expiration_ms // 1000,
    }
    settings = get_settings()
    return jwt.encode(payload, settings.jwt_secret, algorithm=ALGORITHM)


def generate_access_token(username: str, role: str) -> str:
    settings = get_settings()
    return _build_token(username, role, TYPE_ACCESS, settings.access_token_expiration_ms)


def generate_refresh_token(username: str, role: str) -> str:
    settings = get_settings()
    return _build_token(username, role, TYPE_REFRESH, settings.refresh_token_expiration_ms)


def decode_token(token: str) -> Optional[dict]:
    settings = get_settings()
    try:
        return jwt.decode(token, settings.jwt_secret, algorithms=[ALGORITHM])
    except jwt.PyJWTError:
        return None


def is_refresh_token(token: str) -> bool:
    claims = decode_token(token)
    return bool(claims) and claims.get("type") == TYPE_REFRESH


class CurrentUser:
    def __init__(self, username: str, role: str):
        self.username = username
        self.role = role


def _extract_token(authorization: Optional[str]) -> Optional[str]:
    if authorization and authorization.startswith("Bearer "):
        return authorization[len("Bearer "):]
    return None


def get_current_user(
    authorization: Optional[str] = Header(default=None),
    db: Session = Depends(get_db),
) -> CurrentUser:
    """FastAPI dependency requiring a valid access-token JWT (instructions section 11A)."""
    token = _extract_token(authorization)
    if not token:
        raise UnauthorizedException()

    claims = decode_token(token)
    if not claims or claims.get("type") != TYPE_ACCESS:
        raise UnauthorizedException()

    username = claims.get("sub")
    user = db.query(User).filter(User.username.ilike(username)).first()
    if not user or user.status != "ACTIVE":
        raise UnauthorizedException()

    return CurrentUser(username=user.username, role=user.role)


def require_roles(*roles: str):
    """FastAPI dependency factory mirroring @PreAuthorize("hasAnyRole(...)")/hasRole(...)."""

    def dependency(current_user: CurrentUser = Depends(get_current_user)) -> CurrentUser:
        if current_user.role not in roles:
            raise AccessDeniedException()
        return current_user

    return dependency


require_any_role = require_roles(*Roles.ALL)
require_admin_or_editor = require_roles(*Roles.ADMIN_EDITOR)
require_admin = require_roles(Roles.ADMIN)

-------------------------------------------------------------
C:\ws\agent\website-college\backend-python\app\seed.py
"""Startup schema creation + sample data seeding.

Mirrors schema.sql, data.sql, and config/DataInitializer.java (instructions
sections 9, 10). Runs once per process against the shared in-memory SQLite
database, since the data has the same process-lifetime as H2 `mem:` did.
"""
from __future__ import annotations

from datetime import datetime, timedelta, timezone

from sqlalchemy.orm import Session

from . import models
from .database import Base, SessionLocal, engine
from .security import hash_password

COURSES = [
    ("BCA", "Bachelor of Computer Applications", "3 Years", "10+2 with Mathematics", "120000.00",
     "Undergraduate program focused on computer applications and software development.",
     60, "Software Developer, System Analyst, IT Consultant"),
    ("MCA", "Master of Computer Applications", "2 Years", "Bachelor's degree with Mathematics", "150000.00",
     "Postgraduate program for advanced computer science and application development.",
     60, "Senior Software Engineer, Architect, Tech Lead"),
    ("BBA", "Bachelor of Business Administration", "3 Years", "10+2 in any stream", "100000.00",
     "Undergraduate program covering core business and management concepts.",
     80, "Business Analyst, HR Executive, Marketing Executive"),
    ("MBA", "Master of Business Administration", "2 Years", "Bachelor's degree in any discipline", "200000.00",
     "Postgraduate management program with specializations in Finance, HR, and Marketing.",
     80, "Manager, Business Consultant, Entrepreneur"),
    ("BA", "Bachelor of Arts", "3 Years", "10+2 in any stream", "60000.00",
     "Undergraduate program in humanities and social sciences.",
     100, "Civil Services, Teaching, Journalism"),
    ("MA", "Master of Arts", "2 Years", "Bachelor's degree in relevant discipline", "80000.00",
     "Postgraduate program in humanities and social sciences.",
     60, "Researcher, Professor, Content Writer"),
    ("BCOM", "Bachelor of Commerce", "3 Years", "10+2 in Commerce/any stream", "70000.00",
     "Undergraduate program in commerce, accounting, and finance.",
     100, "Accountant, Tax Consultant, Bank PO"),
    ("MCOM", "Master of Commerce", "2 Years", "Bachelor's degree in Commerce", "90000.00",
     "Postgraduate program in advanced commerce and finance.",
     60, "Financial Analyst, Auditor, Professor"),
]

DEPARTMENTS = ["Computer Applications", "Management Studies", "Commerce", "Arts and Humanities"]
DESIGNATIONS = ["Professor", "Associate Professor", "Assistant Professor"]
QUALIFICATIONS = ["Ph.D", "M.Tech / M.Phil"]

SUBJECTS = ["Admission Query", "Fee Structure Query", "General Query"]

CITIES = ["Springfield", "Riverdale", "Fairview", "Lakeside", "Greenville"]
STATES = ["Telangana", "Andhra Pradesh", "Karnataka", "Maharashtra", "Tamil Nadu"]
COURSE_CODES_CYCLE = ["BCA", "MCA", "BBA", "MBA", "BA", "MA", "BCOM", "MCOM"]
INQUIRY_STATUSES = ["NEW", "CONTACTED", "INTERESTED", "VISITED", "ADMITTED", "CLOSED"]


def _now() -> datetime:
    return datetime.now(timezone.utc).replace(tzinfo=None)


def _seed_college_info(db: Session) -> None:
    if db.query(models.CollegeInfo).count() > 0:
        return
    db.add(models.CollegeInfo(
        name="National Institute of Advanced Studies",
        vision="To be a globally recognized center of academic excellence and innovation.",
        mission="To empower students with knowledge, skills, and values to excel in a competitive world.",
        about_us=(
            "Established in 1985, the college has been a pioneer in higher education, offering "
            "undergraduate and postgraduate programs across commerce, management, and computer applications."
        ),
        address="123 College Road, Knowledge City, Springfield, 500001",
        phone="+91-40-12345678",
        email="info@collegeportal.edu",
        website="https://www.collegeportal.edu",
    ))


def _seed_courses(db: Session) -> None:
    if db.query(models.Course).count() > 0:
        return
    for code, name, duration, eligibility, fees, description, intake, careers in COURSES:
        db.add(models.Course(
            course_code=code,
            course_name=name,
            duration=duration,
            eligibility=eligibility,
            fees=fees,
            description=description,
            intake_capacity=intake,
            career_opportunities=careers,
            status="ACTIVE",
        ))


def _seed_faculty(db: Session) -> None:
    if db.query(models.Faculty).count() > 0:
        return
    for x in range(1, 21):
        db.add(models.Faculty(
            name=f"Faculty Member {x}",
            department=DEPARTMENTS[x % 4],
            designation=DESIGNATIONS[x % 3],
            qualification=QUALIFICATIONS[x % 2],
            experience=5 + (x % 20),
            specialization=f"Specialization Area {x % 6}",
            photo_url=f"https://cdn.collegeportal.edu/faculty/faculty-{x}.jpg",
        ))


def _seed_contacts(db: Session) -> None:
    if db.query(models.ContactUs).count() > 0:
        return
    now = _now()
    for x in range(1, 21):
        db.add(models.ContactUs(
            name=f"Visitor {x}",
            email=f"visitor{x}@example.com",
            mobile=f"90000{x:05d}",
            subject=SUBJECTS[x % 3],
            message=f"This is a sample contact message number {x}.",
            created_date=now - timedelta(days=x),
        ))


def _seed_admission_inquiries(db: Session) -> None:
    if db.query(models.AdmissionInquiry).count() > 0:
        return
    now = _now()
    for x in range(1, 101):
        db.add(models.AdmissionInquiry(
            inquiry_number=f"INQ-2026-{x:06d}",
            student_name=f"Student {x}",
            mobile=f"98{x:08d}",
            email=f"student{x}@example.com",
            city=CITIES[x % 5],
            state=STATES[x % 5],
            course_interest=COURSE_CODES_CYCLE[x % 8],
            qualification="10+2" if x % 2 == 0 else "Bachelor's Degree",
            remarks=f"Auto-generated sample inquiry {x}",
            status=INQUIRY_STATUSES[x % 6],
            created_date=now - timedelta(days=x % 90),
        ))


def _seed_users(db: Session) -> None:
    """Seeds the 3 default users with BCrypt-encoded passwords (section 10)."""
    default_users = [
        ("admin", "admin123", "ROLE_ADMIN"),
        ("editor", "editor123", "ROLE_EDITOR"),
        ("viewer", "viewer123", "ROLE_VIEWER"),
    ]
    for username, raw_password, role in default_users:
        exists = db.query(models.User).filter(models.User.username.ilike(username)).first()
        if exists:
            continue
        db.add(models.User(
            username=username,
            password=hash_password(raw_password),
            role=role,
            status="ACTIVE",
        ))


def seed_all() -> None:
    Base.metadata.create_all(bind=engine)
    db = SessionLocal()
    try:
        _seed_college_info(db)
        _seed_courses(db)
        _seed_faculty(db)
        _seed_contacts(db)
        _seed_admission_inquiries(db)
        _seed_users(db)
        db.commit()
    finally:
        db.close()

-------------------------------------------------------------
C:\ws\agent\website-college\backend-python\.env.example
# Copy to .env (or set as real environment variables) for local runs.
APP_PROFILE=local
PORT=8080
JWT_SECRET=local-dev-only-change-me-must-be-32-bytes-minimum-secret
CORS_ALLOWED_ORIGINS=http://localhost:4200

# Optional: point at a real PostgreSQL database instead of the default
# in-memory SQLite. Tables + sample data are created automatically on
# startup either way (see app/seed.py). Requires psycopg2-binary (already
# in requirements.txt).
# DATABASE_URL=postgresql+psycopg2://college_user:college_pass@localhost:5432/collegedb

-------------------------------------------------------------
C:\ws\agent\website-college\backend-python\Dockerfile
# College Digital Foundation Portal - Backend (Python/FastAPI)
FROM python:3.12-slim

WORKDIR /app

RUN addgroup --system app && adduser --system --ingroup app app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app ./app

USER app
ENV APP_PROFILE=local
EXPOSE 8080

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8080"]

-------------------------------------------------------------
C:\ws\agent\website-college\backend-python\requirements.txt
fastapi==0.115.0
uvicorn[standard]==0.30.6
sqlalchemy==2.0.35
pydantic==2.9.2
email-validator==2.2.0
PyJWT==2.9.0
bcrypt==4.2.0
python-multipart==0.0.9
psycopg2-binary==2.9.9
===========================================================================
C:\ws\agent\website-college\scripts\postgres_schema_and_seed.sql
-- Creates the College Digital Foundation Portal schema in PostgreSQL and
-- seeds it with the same sample data the app generates automatically for
-- SQLite (see backend-python/app/seed.py). Table/column names match the
-- SQLAlchemy models exactly (backend-python/app/models.py), so the FastAPI
-- app can be pointed at this same database via DATABASE_URL with no code
-- changes.
--
-- Usage:
--   createdb collegedb
--   psql -d collegedb -f scripts/postgres_schema_and_seed.sql
--
-- Note: the 3 default users (admin/editor/viewer) are intentionally NOT
-- created here, since their passwords must be BCrypt-hashed and no
-- plaintext/precomputed hash is committed to source control. After running
-- this script, either:
--   (a) start the app once with DATABASE_URL pointing at this database -
--       app/seed.py will create the `users` table rows with BCrypt hashes, or
--   (b) run `python scripts/seed_postgres_users.py` (see that file).

BEGIN;

CREATE TABLE IF NOT EXISTS college_info (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    vision VARCHAR(2000),
    mission VARCHAR(2000),
    about_us TEXT,
    address VARCHAR(500),
    phone VARCHAR(50),
    email VARCHAR(255),
    website VARCHAR(255)
);

CREATE TABLE IF NOT EXISTS course (
    id BIGSERIAL PRIMARY KEY,
    course_code VARCHAR(50) NOT NULL UNIQUE,
    course_name VARCHAR(255) NOT NULL,
    duration VARCHAR(100),
    eligibility VARCHAR(500),
    fees NUMERIC(12,2),
    description TEXT,
    intake_capacity INTEGER,
    career_opportunities VARCHAR(1000),
    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE'
);

CREATE TABLE IF NOT EXISTS faculty (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    department VARCHAR(255),
    designation VARCHAR(255),
    qualification VARCHAR(255),
    experience INTEGER,
    specialization VARCHAR(500),
    photo_url VARCHAR(500)
);

CREATE TABLE IF NOT EXISTS contact_us (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL,
    mobile VARCHAR(20),
    subject VARCHAR(255),
    message TEXT,
    created_date TIMESTAMP NOT NULL DEFAULT now()
);

CREATE TABLE IF NOT EXISTS admission_inquiry (
    id BIGSERIAL PRIMARY KEY,
    inquiry_number VARCHAR(50) NOT NULL UNIQUE,
    student_name VARCHAR(255) NOT NULL,
    mobile VARCHAR(20) NOT NULL,
    email VARCHAR(255),
    city VARCHAR(100),
    state VARCHAR(100),
    course_interest VARCHAR(255),
    qualification VARCHAR(255),
    remarks VARCHAR(2000),
    status VARCHAR(50) NOT NULL DEFAULT 'NEW',
    created_date TIMESTAMP NOT NULL DEFAULT now()
);

CREATE TABLE IF NOT EXISTS users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(255) NOT NULL UNIQUE,
    password VARCHAR(500) NOT NULL,
    role VARCHAR(50) NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE'
);

-- 1 college record
INSERT INTO college_info (name, vision, mission, about_us, address, phone, email, website)
SELECT
    'National Institute of Advanced Studies',
    'To be a globally recognized center of academic excellence and innovation.',
    'To empower students with knowledge, skills, and values to excel in a competitive world.',
    'Established in 1985, the college has been a pioneer in higher education, offering ' ||
        'undergraduate and postgraduate programs across commerce, management, and computer applications.',
    '123 College Road, Knowledge City, Springfield, 500001',
    '+91-40-12345678',
    'info@collegeportal.edu',
    'https://www.collegeportal.edu'
WHERE NOT EXISTS (SELECT 1 FROM college_info);

-- 8 courses
INSERT INTO course (course_code, course_name, duration, eligibility, fees, description, intake_capacity, career_opportunities, status)
SELECT v.course_code, v.course_name, v.duration, v.eligibility, v.fees, v.description, v.intake_capacity, v.career_opportunities, 'ACTIVE'
FROM (VALUES
    ('BCA', 'Bachelor of Computer Applications', '3 Years', '10+2 with Mathematics', 120000.00,
     'Undergraduate program focused on computer applications and software development.',
     60, 'Software Developer, System Analyst, IT Consultant'),
    ('MCA', 'Master of Computer Applications', '2 Years', 'Bachelor''s degree with Mathematics', 150000.00,
     'Postgraduate program for advanced computer science and application development.',
     60, 'Senior Software Engineer, Architect, Tech Lead'),
    ('BBA', 'Bachelor of Business Administration', '3 Years', '10+2 in any stream', 100000.00,
     'Undergraduate program covering core business and management concepts.',
     80, 'Business Analyst, HR Executive, Marketing Executive'),
    ('MBA', 'Master of Business Administration', '2 Years', 'Bachelor''s degree in any discipline', 200000.00,
     'Postgraduate management program with specializations in Finance, HR, and Marketing.',
     80, 'Manager, Business Consultant, Entrepreneur'),
    ('BA', 'Bachelor of Arts', '3 Years', '10+2 in any stream', 60000.00,
     'Undergraduate program in humanities and social sciences.',
     100, 'Civil Services, Teaching, Journalism'),
    ('MA', 'Master of Arts', '2 Years', 'Bachelor''s degree in relevant discipline', 80000.00,
     'Postgraduate program in humanities and social sciences.',
     60, 'Researcher, Professor, Content Writer'),
    ('BCOM', 'Bachelor of Commerce', '3 Years', '10+2 in Commerce/any stream', 70000.00,
     'Undergraduate program in commerce, accounting, and finance.',
     100, 'Accountant, Tax Consultant, Bank PO'),
    ('MCOM', 'Master of Commerce', '2 Years', 'Bachelor''s degree in Commerce', 90000.00,
     'Postgraduate program in advanced commerce and finance.',
     60, 'Financial Analyst, Auditor, Professor')
) AS v(course_code, course_name, duration, eligibility, fees, description, intake_capacity, career_opportunities)
WHERE NOT EXISTS (SELECT 1 FROM course);

-- 20 faculty
INSERT INTO faculty (name, department, designation, qualification, experience, specialization, photo_url)
SELECT
    'Faculty Member ' || x,
    (ARRAY['Computer Applications', 'Management Studies', 'Commerce', 'Arts and Humanities'])[(x % 4) + 1],
    (ARRAY['Professor', 'Associate Professor', 'Assistant Professor'])[(x % 3) + 1],
    (ARRAY['Ph.D', 'M.Tech / M.Phil'])[(x % 2) + 1],
    5 + (x % 20),
    'Specialization Area ' || (x % 6),
    'https://cdn.collegeportal.edu/faculty/faculty-' || x || '.jpg'
FROM generate_series(1, 20) AS x
WHERE NOT EXISTS (SELECT 1 FROM faculty);

-- 20 contact requests
INSERT INTO contact_us (name, email, mobile, subject, message, created_date)
SELECT
    'Visitor ' || x,
    'visitor' || x || '@example.com',
    '90000' || lpad(x::text, 5, '0'),
    (ARRAY['Admission Query', 'Fee Structure Query', 'General Query'])[(x % 3) + 1],
    'This is a sample contact message number ' || x || '.',
    now() - (x || ' days')::interval
FROM generate_series(1, 20) AS x
WHERE NOT EXISTS (SELECT 1 FROM contact_us);

-- 100 admission inquiries
INSERT INTO admission_inquiry (inquiry_number, student_name, mobile, email, city, state, course_interest, qualification, remarks, status, created_date)
SELECT
    'INQ-2026-' || lpad(x::text, 6, '0'),
    'Student ' || x,
    '98' || lpad(x::text, 8, '0'),
    'student' || x || '@example.com',
    (ARRAY['Springfield', 'Riverdale', 'Fairview', 'Lakeside', 'Greenville'])[(x % 5) + 1],
    (ARRAY['Telangana', 'Andhra Pradesh', 'Karnataka', 'Maharashtra', 'Tamil Nadu'])[(x % 5) + 1],
    (ARRAY['BCA', 'MCA', 'BBA', 'MBA', 'BA', 'MA', 'BCOM', 'MCOM'])[(x % 8) + 1],
    CASE WHEN x % 2 = 0 THEN '10+2' ELSE 'Bachelor''s Degree' END,
    'Auto-generated sample inquiry ' || x,
    (ARRAY['NEW', 'CONTACTED', 'INTERESTED', 'VISITED', 'ADMITTED', 'CLOSED'])[(x % 6) + 1],
    now() - ((x % 90) || ' days')::interval
FROM generate_series(1, 100) AS x
WHERE NOT EXISTS (SELECT 1 FROM admission_inquiry);

COMMIT;

-------------------------------------------------------------
C:\ws\agent\website-college\scripts\seed_postgres_users.py
"""Seeds the 3 default application users (admin/editor/viewer) into a
PostgreSQL database, using the app's own BCrypt hashing (app/security.py) so
credentials are identical to what app/seed.py creates automatically for the
default in-memory SQLite database. Run this after
scripts/postgres_schema_and_seed.sql if you provisioned the schema/sample
data directly via psql instead of starting the app once against Postgres.

Usage (PowerShell):
    $env:DATABASE_URL = "postgresql+psycopg2://college_user:college_pass@localhost:5432/collegedb"
    pip install -r backend-python/requirements.txt
    python scripts/seed_postgres_users.py
"""
import os
import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).resolve().parent.parent / "backend-python"))

from app.database import SessionLocal  # noqa: E402
from app.models import User  # noqa: E402
from app.security import hash_password  # noqa: E402

DEFAULT_USERS = [
    ("admin", "admin123", "ROLE_ADMIN"),
    ("editor", "editor123", "ROLE_EDITOR"),
    ("viewer", "viewer123", "ROLE_VIEWER"),
]


def main() -> None:
    if not os.getenv("DATABASE_URL"):
        raise SystemExit("Set DATABASE_URL to your PostgreSQL connection string before running this script.")

    db = SessionLocal()
    try:
        for username, raw_password, role in DEFAULT_USERS:
            if db.query(User).filter(User.username.ilike(username)).first():
                print(f"Skipping {username} (already exists)")
                continue
            db.add(User(username=username, password=hash_password(raw_password), role=role, status="ACTIVE"))
            print(f"Created {username} ({role})")
        db.commit()
    finally:
        db.close()


if __name__ == "__main__":
    main()

-------------------------------------------------------------
C:\ws\agent\website-college\scripts\README.md
# Scripts

Utility scripts for local development and operations (e.g. database reset,
bulk data generation, log tailing) go here. None are required to build, test,
or run the project today — add scripts as operational needs arise.

## PostgreSQL schema + sample data

The app defaults to an in-memory SQLite database and creates/seeds its own
tables on startup (`backend-python/app/seed.py`). To run against a real
PostgreSQL database instead:

**Option A — let the app do it (recommended):**
```powershell
$env:DATABASE_URL = "postgresql+psycopg2://college_user:college_pass@localhost:5432/collegedb"
cd backend-python
pip install -r requirements.txt
uvicorn app.main:app --reload --port 8080
```
Tables and all sample data (including BCrypt-hashed admin/editor/viewer
users) are created automatically on first startup, same as SQLite.

**Option B — provision the database directly via psql:**
```powershell
createdb collegedb
psql -d collegedb -f scripts/postgres_schema_and_seed.sql
$env:DATABASE_URL = "postgresql+psycopg2://college_user:college_pass@localhost:5432/collegedb"
python scripts/seed_postgres_users.py   # adds admin/editor/viewer with BCrypt hashes
```
`postgres_schema_and_seed.sql` creates all tables and seeds college info,
courses, faculty, contacts, and admission inquiries (matching counts/values
in `app/seed.py`). It intentionally skips the `users` table — passwords must
be BCrypt-hashed, so `seed_postgres_users.py` creates them using the app's
own `hash_password()` instead of a committed plaintext/hash.
============================================================
C:\ws\agent\website-college\frontend\src\app\core\constants\roles.constant.ts
export const Roles = {
  ADMIN: 'ROLE_ADMIN',
  EDITOR: 'ROLE_EDITOR',
  VIEWER: 'ROLE_VIEWER'
} as const;

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\core\guards\auth.guard.spec.ts
import { TestBed } from '@angular/core/testing';
import { Router, UrlTree } from '@angular/router';
import { RouterTestingModule } from '@angular/router/testing';
import { authGuard } from './auth.guard';
import { AuthService } from '../services/auth.service';

describe('authGuard', () => {
  let authServiceSpy: jasmine.SpyObj<AuthService>;

  beforeEach(() => {
    authServiceSpy = jasmine.createSpyObj('AuthService', ['isAuthenticated']);
    TestBed.configureTestingModule({
      imports: [RouterTestingModule],
      providers: [{ provide: AuthService, useValue: authServiceSpy }]
    });
  });

  it('should allow navigation when authenticated', () => {
    authServiceSpy.isAuthenticated.and.returnValue(true);

    const result = TestBed.runInInjectionContext(() =>
      authGuard({} as any, { url: '/dashboard' } as any)
    );

    expect(result).toBeTrue();
  });

  it('should redirect to login when not authenticated', () => {
    authServiceSpy.isAuthenticated.and.returnValue(false);

    const result = TestBed.runInInjectionContext(() =>
      authGuard({} as any, { url: '/dashboard' } as any)
    ) as UrlTree;

    const router = TestBed.inject(Router);
    expect(result.toString()).toBe(router.createUrlTree(['/login'], { queryParams: { returnUrl: '/dashboard' } }).toString());
  });
});

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\core\guards\auth.guard.ts
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';
import { AuthService } from '../services/auth.service';

/**
 * Blocks access to protected routes when the user is not authenticated
 * (instructions section 13A - functional route guards).
 */
export const authGuard: CanActivateFn = (route, state) => {
  const authService = inject(AuthService);
  const router = inject(Router);

  if (authService.isAuthenticated()) {
    return true;
  }

  return router.createUrlTree(['/login'], { queryParams: { returnUrl: state.url } });
};

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\core\guards\role.guard.ts
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';
import { AuthService } from '../services/auth.service';

/**
 * Restricts access to routes based on the `roles` array declared in route
 * data, e.g. `data: { roles: [Roles.ADMIN] }` (instructions section 13A).
 */
export const roleGuard: CanActivateFn = (route) => {
  const authService = inject(AuthService);
  const router = inject(Router);
  const requiredRoles = (route.data['roles'] as string[] | undefined) ?? [];

  if (requiredRoles.length === 0 || authService.hasAnyRole(requiredRoles)) {
    return true;
  }

  return router.createUrlTree(['/dashboard']);
};

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\core\interceptors\auth.interceptor.ts
import { HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';
import { AuthService } from '../services/auth.service';

const AUTH_EXEMPT_PATHS = ['/auth/login', '/auth/refresh', '/auth/forgot-password', '/auth/reset-password'];

/**
 * Attaches the JWT access token to outgoing requests (instructions section 13A).
 */
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const authService = inject(AuthService);
  const isExempt = AUTH_EXEMPT_PATHS.some(path => req.url.includes(path));
  const token = authService.getAccessToken();

  if (!isExempt && token) {
    req = req.clone({ setHeaders: { Authorization: `Bearer ${token}` } });
  }

  return next(req);
};

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\core\interceptors\error.interceptor.ts
import { HttpErrorResponse, HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';
import { Router } from '@angular/router';
import { catchError, throwError } from 'rxjs';
import { ApiErrorResponse } from '../models/api-error.model';
import { AuthService } from '../services/auth.service';
import { NotificationService } from '../services/notification.service';

/**
 * Unwraps the standard API error envelope (instructions section 4B) and
 * surfaces user-friendly messages. Redirects to login on 401 (instructions
 * section 13A).
 */
export const errorInterceptor: HttpInterceptorFn = (req, next) => {
  const router = inject(Router);
  const authService = inject(AuthService);
  const notificationService = inject(NotificationService);

  return next(req).pipe(
    catchError((error: HttpErrorResponse) => {
      const apiError = error.error as ApiErrorResponse | undefined;
      const baseMessage = apiError?.message ?? 'An unexpected error occurred. Please try again.';
      const message = apiError?.details?.length ? `${baseMessage}: ${apiError.details.join('; ')}` : baseMessage;

      if (error.status === 401) {
        authService.logout();
        notificationService.error('Your session has expired. Please log in again.');
        router.navigate(['/login']);
      } else if (error.status === 403) {
        notificationService.error('You do not have permission to perform this action.');
      } else if (error.status === 0) {
        notificationService.error('Unable to reach the server. Please check your connection.');
      } else {
        notificationService.error(message);
      }

      return throwError(() => error);
    })
  );
};

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\core\models\admission-inquiry.model.ts
export type InquiryStatus = 'NEW' | 'CONTACTED' | 'INTERESTED' | 'VISITED' | 'ADMITTED' | 'CLOSED';

export interface AdmissionInquiryRequest {
  studentName: string;
  mobile: string;
  email: string;
  city: string;
  state: string;
  courseInterest: string;
  qualification: string;
  remarks: string;
}

export interface AdmissionInquiry {
  id: number;
  inquiryNumber: string;
  studentName: string;
  mobile: string;
  email: string;
  city: string;
  state: string;
  courseInterest: string;
  qualification: string;
  remarks: string;
  status: InquiryStatus;
  createdDate: string;
}

export interface AdmissionInquiryStatusUpdateRequest {
  status: InquiryStatus | string;
  remarks: string;
}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\core\models\api-error.model.ts
/**
 * Standard error response envelope. Mirrors backend
 * com.college.portal.dto.common.ErrorResponse (instructions 4B).
 */
export interface ApiErrorResponse {
  success: boolean;
  errorCode: string;
  message: string;
  details: string[];
  timestamp: string;
}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\core\models\api-response.model.ts
/**
 * Standard success response envelope returned by every backend endpoint.
 * Mirrors backend com.college.portal.dto.common.ApiResponse (instructions 4B).
 */
export interface ApiResponse<T> {
  success: boolean;
  data: T;
  message: string;
  timestamp: string;
}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\core\models\auth.model.ts
export interface LoginRequest {
  username: string;
  password: string;
}

export interface LoginResponse {
  accessToken: string;
  refreshToken: string;
  tokenType: string;
  username: string;
  role: string;
  expiresInMs: number;
}

export interface RefreshTokenRequest {
  refreshToken: string;
}

export interface ForgotPasswordRequest {
  usernameOrEmail: string;
}

export interface ResetPasswordRequest {
  token: string;
  newPassword: string;
}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\core\models\college-info.model.ts
export interface CollegeInfo {
  id: number;
  name: string;
  vision: string;
  mission: string;
  aboutUs: string;
  address: string;
  phone: string;
  email: string;
  website: string;
}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\core\models\contact.model.ts
export interface ContactUsRequest {
  name: string;
  mobile: string;
  email: string;
  subject: string;
  message: string;
}

export interface ContactUs extends ContactUsRequest {
  id: number;
  createdDate: string;
}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\core\models\course.model.ts
export interface Course {
  id: number;
  courseCode: string;
  courseName: string;
  duration: string;
  eligibility: string;
  fees: number;
  description: string;
  intakeCapacity: number;
  careerOpportunities: string;
  status: 'ACTIVE' | 'INACTIVE';
}

export interface CourseRequest {
  courseCode: string;
  courseName: string;
  duration: string;
  eligibility: string;
  fees: number;
  description: string;
  intakeCapacity: number;
  careerOpportunities: string;
  status: string;
}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\core\models\dashboard.model.ts
export interface ChartPoint {
  label: string;
  value: number;
}

export interface DashboardSummary {
  totalCourses: number;
  totalFaculty: number;
  totalAdmissionInquiries: number;
  totalContacts: number;
  monthlyInquiries: ChartPoint[];
  courseWiseInquiries: ChartPoint[];
  dailyRequests: ChartPoint[];
}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\core\models\faculty.model.ts
export interface Faculty {
  id: number;
  name: string;
  department: string;
  designation: string;
  qualification: string;
  experience: number;
  specialization: string;
  photoUrl: string;
}

export interface FacultyRequest {
  name: string;
  department: string;
  designation: string;
  qualification: string;
  experience: number;
  specialization: string;
  photoUrl: string;
}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\core\models\fee.model.ts
export interface CourseFee {
  courseCode: string;
  courseName: string;
  semesterFees: number;
}

export interface FeeStructure {
  courseFees: CourseFee[];
  admissionFees: number;
  hostelFees: number;
  transportFees: number;
  scholarshipInformation: string;
}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\core\models\home.model.ts
export interface HomeResponse {
  collegeName: string;
  logoUrl: string;
  vision: string;
  mission: string;
  principalMessage: string;
  latestNews: string[];
  upcomingEvents: string[];
  collegeHighlights: string[];
  placementHighlights: string[];
  admissionBannerText: string;
}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\core\models\page-response.model.ts
/**
 * Standard pagination wrapper for list endpoints. Mirrors backend
 * com.college.portal.dto.common.PageResponse (instructions 4B).
 */
export interface PageResponse<T> {
  content: T[];
  totalElements: number;
  totalPages: number;
  page: number;
  size: number;
}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\core\services\admission.service.ts
import { HttpClient, HttpParams } from '@angular/common/http';
import { Injectable } from '@angular/core';
import { Observable } from 'rxjs';
import { environment } from '../../../environments/environment';
import { ApiResponse } from '../models/api-response.model';
import { PageResponse } from '../models/page-response.model';
import {
  AdmissionInquiry,
  AdmissionInquiryRequest,
  AdmissionInquiryStatusUpdateRequest
} from '../models/admission-inquiry.model';

@Injectable({ providedIn: 'root' })
export class AdmissionService {
  private readonly baseUrl = `${environment.apiUrl}/admissions/inquiry`;
  private readonly adminUrl = `${environment.apiUrl}/admin/admissions`;

  constructor(private readonly http: HttpClient) {}

  submitInquiry(request: AdmissionInquiryRequest): Observable<ApiResponse<AdmissionInquiry>> {
    return this.http.post<ApiResponse<AdmissionInquiry>>(this.baseUrl, request);
  }

  getInquiryById(id: number): Observable<ApiResponse<AdmissionInquiry>> {
    return this.http.get<ApiResponse<AdmissionInquiry>>(`${this.baseUrl}/${id}`);
  }

  getAllInquiries(page = 0, size = 10): Observable<ApiResponse<PageResponse<AdmissionInquiry>>> {
    const params = new HttpParams().set('page', page).set('size', size);
    return this.http.get<ApiResponse<PageResponse<AdmissionInquiry>>>(this.adminUrl, { params });
  }

  searchInquiries(keyword: string, page = 0, size = 10): Observable<ApiResponse<PageResponse<AdmissionInquiry>>> {
    const params = new HttpParams().set('keyword', keyword).set('page', page).set('size', size);
    return this.http.get<ApiResponse<PageResponse<AdmissionInquiry>>>(`${this.adminUrl}/search`, { params });
  }

  getInquiriesByStatus(status: string, page = 0, size = 10): Observable<ApiResponse<PageResponse<AdmissionInquiry>>> {
    const params = new HttpParams().set('page', page).set('size', size);
    return this.http.get<ApiResponse<PageResponse<AdmissionInquiry>>>(`${this.adminUrl}/status/${status}`, { params });
  }

  updateStatus(id: number, request: AdmissionInquiryStatusUpdateRequest): Observable<ApiResponse<AdmissionInquiry>> {
    return this.http.put<ApiResponse<AdmissionInquiry>>(`${this.adminUrl}/${id}/status`, request);
  }

  exportCsvUrl(): string {
    return `${this.adminUrl}/export`;
  }
}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\core\services\auth.service.spec.ts
import { TestBed } from '@angular/core/testing';
import { HttpClientTestingModule, HttpTestingController } from '@angular/common/http/testing';
import { AuthService } from './auth.service';
import { environment } from '../../../environments/environment';
import { LoginResponse } from '../models/auth.model';
import { ApiResponse } from '../models/api-response.model';

describe('AuthService', () => {
  let service: AuthService;
  let httpMock: HttpTestingController;

  beforeEach(() => {
    localStorage.clear();
    TestBed.configureTestingModule({
      imports: [HttpClientTestingModule],
      providers: [AuthService]
    });
    service = TestBed.inject(AuthService);
    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => {
    httpMock.verify();
    localStorage.clear();
  });

  it('should be created', () => {
    expect(service).toBeTruthy();
  });

  it('should not be authenticated initially', () => {
    expect(service.isAuthenticated()).toBeFalse();
  });

  it('should store tokens and mark the user authenticated after login', () => {
    const mockResponse: ApiResponse<LoginResponse> = {
      success: true,
      message: 'Login successful',
      timestamp: new Date().toISOString(),
      data: {
        accessToken: 'access-token',
        refreshToken: 'refresh-token',
        tokenType: 'Bearer',
        username: 'admin',
        role: 'ROLE_ADMIN',
        expiresInMs: 3600000
      }
    };

    service.login({ username: 'admin', password: 'admin123' }).subscribe(response => {
      expect(response.data.accessToken).toBe('access-token');
    });

    const req = httpMock.expectOne(`${environment.apiUrl}/auth/login`);
    expect(req.request.method).toBe('POST');
    req.flush(mockResponse);

    expect(service.isAuthenticated()).toBeTrue();
    expect(service.getAccessToken()).toBe('access-token');
    expect(service.hasAnyRole(['ROLE_ADMIN'])).toBeTrue();
    expect(service.hasAnyRole(['ROLE_VIEWER'])).toBeFalse();
  });

  it('should clear stored tokens on logout', () => {
    localStorage.setItem('cp_access_token', 'token');
    const httpTestingController = TestBed.inject(HttpTestingController);

    service.logout();
    httpTestingController.expectOne(`${environment.apiUrl}/auth/logout`).flush({});

    expect(service.isAuthenticated()).toBeFalse();
    expect(service.getAccessToken()).toBeNull();
  });
});

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\core\services\auth.service.ts
import { HttpClient } from '@angular/common/http';
import { Injectable } from '@angular/core';
import { BehaviorSubject, Observable, tap } from 'rxjs';
import { environment } from '../../../environments/environment';
import { ApiResponse } from '../models/api-response.model';
import {
  ForgotPasswordRequest,
  LoginRequest,
  LoginResponse,
  RefreshTokenRequest,
  ResetPasswordRequest
} from '../models/auth.model';

const ACCESS_TOKEN_KEY = 'cp_access_token';
const REFRESH_TOKEN_KEY = 'cp_refresh_token';
const USERNAME_KEY = 'cp_username';
const ROLE_KEY = 'cp_role';

export interface CurrentUser {
  username: string;
  role: string;
}

@Injectable({ providedIn: 'root' })
export class AuthService {
  private readonly baseUrl = `${environment.apiUrl}/auth`;

  private readonly currentUserSubject = new BehaviorSubject<CurrentUser | null>(this.readStoredUser());
  readonly currentUser$ = this.currentUserSubject.asObservable();

  constructor(private readonly http: HttpClient) {}

  login(request: LoginRequest): Observable<ApiResponse<LoginResponse>> {
    return this.http.post<ApiResponse<LoginResponse>>(`${this.baseUrl}/login`, request).pipe(
      tap(response => this.storeSession(response.data))
    );
  }

  refreshToken(): Observable<ApiResponse<LoginResponse>> {
    const request: RefreshTokenRequest = { refreshToken: this.getRefreshToken() ?? '' };
    return this.http.post<ApiResponse<LoginResponse>>(`${this.baseUrl}/refresh`, request).pipe(
      tap(response => this.storeSession(response.data))
    );
  }

  forgotPassword(request: ForgotPasswordRequest): Observable<ApiResponse<void>> {
    return this.http.post<ApiResponse<void>>(`${this.baseUrl}/forgot-password`, request);
  }

  resetPassword(request: ResetPasswordRequest): Observable<ApiResponse<void>> {
    return this.http.post<ApiResponse<void>>(`${this.baseUrl}/reset-password`, request);
  }

  logout(): void {
    this.http.post(`${this.baseUrl}/logout`, {}).subscribe({ error: () => undefined });
    localStorage.removeItem(ACCESS_TOKEN_KEY);
    localStorage.removeItem(REFRESH_TOKEN_KEY);
    localStorage.removeItem(USERNAME_KEY);
    localStorage.removeItem(ROLE_KEY);
    this.currentUserSubject.next(null);
  }

  getAccessToken(): string | null {
    return localStorage.getItem(ACCESS_TOKEN_KEY);
  }

  getRefreshToken(): string | null {
    return localStorage.getItem(REFRESH_TOKEN_KEY);
  }

  isAuthenticated(): boolean {
    return !!this.getAccessToken();
  }

  hasAnyRole(roles: string[]): boolean {
    const current = this.currentUserSubject.value;
    return !!current && roles.includes(current.role);
  }

  private storeSession(data: LoginResponse): void {
    localStorage.setItem(ACCESS_TOKEN_KEY, data.accessToken);
    localStorage.setItem(REFRESH_TOKEN_KEY, data.refreshToken);
    localStorage.setItem(USERNAME_KEY, data.username);
    localStorage.setItem(ROLE_KEY, data.role);
    this.currentUserSubject.next({ username: data.username, role: data.role });
  }

  private readStoredUser(): CurrentUser | null {
    const username = localStorage.getItem(USERNAME_KEY);
    const role = localStorage.getItem(ROLE_KEY);
    return username && role ? { username, role } : null;
  }
}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\core\services\college.service.ts
import { HttpClient } from '@angular/common/http';
import { Injectable } from '@angular/core';
import { Observable } from 'rxjs';
import { environment } from '../../../environments/environment';
import { ApiResponse } from '../models/api-response.model';
import { CollegeInfo } from '../models/college-info.model';

@Injectable({ providedIn: 'root' })
export class CollegeService {
  private readonly baseUrl = `${environment.apiUrl}/college`;

  constructor(private readonly http: HttpClient) {}

  getCollegeInfo(): Observable<ApiResponse<CollegeInfo>> {
    return this.http.get<ApiResponse<CollegeInfo>>(this.baseUrl);
  }
}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\core\services\contact.service.ts
import { HttpClient, HttpParams } from '@angular/common/http';
import { Injectable } from '@angular/core';
import { Observable } from 'rxjs';
import { environment } from '../../../environments/environment';
import { ApiResponse } from '../models/api-response.model';
import { PageResponse } from '../models/page-response.model';
import { ContactUs, ContactUsRequest } from '../models/contact.model';

@Injectable({ providedIn: 'root' })
export class ContactService {
  private readonly baseUrl = `${environment.apiUrl}/contact`;
  private readonly adminUrl = `${environment.apiUrl}/admin/contacts`;

  constructor(private readonly http: HttpClient) {}

  submitContact(request: ContactUsRequest): Observable<ApiResponse<ContactUs>> {
    return this.http.post<ApiResponse<ContactUs>>(this.baseUrl, request);
  }

  getAllContacts(page = 0, size = 10): Observable<ApiResponse<PageResponse<ContactUs>>> {
    const params = new HttpParams().set('page', page).set('size', size);
    return this.http.get<ApiResponse<PageResponse<ContactUs>>>(this.adminUrl, { params });
  }

  getContactById(id: number): Observable<ApiResponse<ContactUs>> {
    return this.http.get<ApiResponse<ContactUs>>(`${this.adminUrl}/${id}`);
  }
}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\core\services\course.service.ts
import { HttpClient, HttpParams } from '@angular/common/http';
import { Injectable } from '@angular/core';
import { Observable } from 'rxjs';
import { environment } from '../../../environments/environment';
import { ApiResponse } from '../models/api-response.model';
import { PageResponse } from '../models/page-response.model';
import { Course, CourseRequest } from '../models/course.model';

@Injectable({ providedIn: 'root' })
export class CourseService {
  private readonly baseUrl = `${environment.apiUrl}/courses`;
  private readonly adminUrl = `${environment.apiUrl}/admin/courses`;

  constructor(private readonly http: HttpClient) {}

  getAllCourses(page = 0, size = 10): Observable<ApiResponse<PageResponse<Course>>> {
    const params = new HttpParams().set('page', page).set('size', size);
    return this.http.get<ApiResponse<PageResponse<Course>>>(this.baseUrl, { params });
  }

  getCourseById(id: number): Observable<ApiResponse<Course>> {
    return this.http.get<ApiResponse<Course>>(`${this.baseUrl}/${id}`);
  }

  searchCourses(name: string, page = 0, size = 10): Observable<ApiResponse<PageResponse<Course>>> {
    const params = new HttpParams().set('name', name).set('page', page).set('size', size);
    return this.http.get<ApiResponse<PageResponse<Course>>>(`${this.adminUrl}/search`, { params });
  }

  createCourse(request: CourseRequest): Observable<ApiResponse<Course>> {
    return this.http.post<ApiResponse<Course>>(this.adminUrl, request);
  }

  updateCourse(id: number, request: CourseRequest): Observable<ApiResponse<Course>> {
    return this.http.put<ApiResponse<Course>>(`${this.adminUrl}/${id}`, request);
  }

  deleteCourse(id: number): Observable<void> {
    return this.http.delete<void>(`${this.adminUrl}/${id}`);
  }
}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\core\services\dashboard.service.ts
import { HttpClient } from '@angular/common/http';
import { Injectable } from '@angular/core';
import { Observable } from 'rxjs';
import { environment } from '../../../environments/environment';
import { ApiResponse } from '../models/api-response.model';
import { DashboardSummary } from '../models/dashboard.model';

@Injectable({ providedIn: 'root' })
export class DashboardService {
  private readonly baseUrl = `${environment.apiUrl}/admin/dashboard`;

  constructor(private readonly http: HttpClient) {}

  getSummary(): Observable<ApiResponse<DashboardSummary>> {
    return this.http.get<ApiResponse<DashboardSummary>>(`${this.baseUrl}/summary`);
  }
}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\core\services\faculty.service.ts
import { HttpClient, HttpParams } from '@angular/common/http';
import { Injectable } from '@angular/core';
import { Observable } from 'rxjs';
import { environment } from '../../../environments/environment';
import { ApiResponse } from '../models/api-response.model';
import { PageResponse } from '../models/page-response.model';
import { Faculty, FacultyRequest } from '../models/faculty.model';

@Injectable({ providedIn: 'root' })
export class FacultyService {
  private readonly baseUrl = `${environment.apiUrl}/faculties`;
  private readonly adminUrl = `${environment.apiUrl}/admin/faculties`;

  constructor(private readonly http: HttpClient) {}

  getAllFaculty(page = 0, size = 10): Observable<ApiResponse<PageResponse<Faculty>>> {
    const params = new HttpParams().set('page', page).set('size', size);
    return this.http.get<ApiResponse<PageResponse<Faculty>>>(this.baseUrl, { params });
  }

  getFacultyById(id: number): Observable<ApiResponse<Faculty>> {
    return this.http.get<ApiResponse<Faculty>>(`${this.baseUrl}/${id}`);
  }

  searchFaculty(keyword: string, page = 0, size = 10): Observable<ApiResponse<PageResponse<Faculty>>> {
    const params = new HttpParams().set('keyword', keyword).set('page', page).set('size', size);
    return this.http.get<ApiResponse<PageResponse<Faculty>>>(`${this.baseUrl}/search`, { params });
  }

  createFaculty(request: FacultyRequest): Observable<ApiResponse<Faculty>> {
    return this.http.post<ApiResponse<Faculty>>(this.adminUrl, request);
  }

  updateFaculty(id: number, request: FacultyRequest): Observable<ApiResponse<Faculty>> {
    return this.http.put<ApiResponse<Faculty>>(`${this.adminUrl}/${id}`, request);
  }

  deleteFaculty(id: number): Observable<void> {
    return this.http.delete<void>(`${this.adminUrl}/${id}`);
  }
}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\core\services\fee.service.ts
import { HttpClient } from '@angular/common/http';
import { Injectable } from '@angular/core';
import { Observable } from 'rxjs';
import { environment } from '../../../environments/environment';
import { ApiResponse } from '../models/api-response.model';
import { FeeStructure } from '../models/fee.model';

@Injectable({ providedIn: 'root' })
export class FeeService {
  private readonly baseUrl = `${environment.apiUrl}/fees`;

  constructor(private readonly http: HttpClient) {}

  getFeeStructure(): Observable<ApiResponse<FeeStructure>> {
    return this.http.get<ApiResponse<FeeStructure>>(this.baseUrl);
  }
}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\core\services\home.service.ts
import { HttpClient } from '@angular/common/http';
import { Injectable } from '@angular/core';
import { Observable } from 'rxjs';
import { environment } from '../../../environments/environment';
import { ApiResponse } from '../models/api-response.model';
import { HomeResponse } from '../models/home.model';

@Injectable({ providedIn: 'root' })
export class HomeService {
  private readonly baseUrl = `${environment.apiUrl}/home`;

  constructor(private readonly http: HttpClient) {}

  getHome(): Observable<ApiResponse<HomeResponse>> {
    return this.http.get<ApiResponse<HomeResponse>>(this.baseUrl);
  }
}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\core\services\notification.service.ts
import { Injectable } from '@angular/core';
import { MatSnackBar } from '@angular/material/snack-bar';

@Injectable({ providedIn: 'root' })
export class NotificationService {
  constructor(private readonly snackBar: MatSnackBar) {}

  success(message: string): void {
    this.snackBar.open(message, 'Close', { duration: 4000, panelClass: 'notification-success' });
  }

  error(message: string): void {
    this.snackBar.open(message, 'Close', { duration: 6000, panelClass: 'notification-error' });
  }
}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\features\about\about.component.ts
import { CommonModule } from '@angular/common';
import { Component, OnInit } from '@angular/core';
import { CollegeService } from '../../core/services/college.service';
import { CollegeInfo } from '../../core/models/college-info.model';
import { LoadingSpinnerComponent } from '../../shared/components/loading-spinner/loading-spinner.component';

@Component({
  selector: 'app-about',
  standalone: true,
  imports: [CommonModule, LoadingSpinnerComponent],
  template: `
    <h2 class="section-title">About the College</h2>
    @if (loading) {
      <app-loading-spinner></app-loading-spinner>
    } @else if (college) {
      <div class="card card-shadow mb-4">
        <div class="card-body">
          <h5 class="card-title">{{ college.name }}</h5>
          <p class="card-text">{{ college.aboutUs }}</p>
        </div>
      </div>
      <div class="row g-4">
        <div class="col-md-6">
          <div class="card card-shadow h-100">
            <div class="card-body">
              <h6 class="card-title">Contact Details</h6>
              <p class="mb-1"><strong>Address:</strong> {{ college.address }}</p>
              <p class="mb-1"><strong>Phone:</strong> {{ college.phone }}</p>
              <p class="mb-1"><strong>Email:</strong> {{ college.email }}</p>
              <p class="mb-0"><strong>Website:</strong> {{ college.website }}</p>
            </div>
          </div>
        </div>
        <div class="col-md-6">
          <div class="card card-shadow h-100">
            <div class="card-body">
              <h6 class="card-title">Vision &amp; Mission</h6>
              <p class="mb-1"><strong>Vision:</strong> {{ college.vision }}</p>
              <p class="mb-0"><strong>Mission:</strong> {{ college.mission }}</p>
            </div>
          </div>
        </div>
      </div>
    } @else {
      <p class="text-muted">College information is not available at the moment.</p>
    }
  `
})
export class AboutComponent implements OnInit {
  college: CollegeInfo | null = null;
  loading = true;

  constructor(private readonly collegeService: CollegeService) {}

  ngOnInit(): void {
    this.collegeService.getCollegeInfo().subscribe({
      next: response => {
        this.college = response.data;
        this.loading = false;
      },
      error: () => (this.loading = false)
    });
  }
}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\features\admissions\admissions.component.ts
import { CommonModule } from '@angular/common';
import { Component, OnInit } from '@angular/core';
import { FormBuilder, ReactiveFormsModule, Validators } from '@angular/forms';
import { AdmissionService } from '../../core/services/admission.service';
import { CourseService } from '../../core/services/course.service';
import { Course } from '../../core/models/course.model';
import { NotificationService } from '../../core/services/notification.service';
import { AdmissionInquiry } from '../../core/models/admission-inquiry.model';

@Component({
  selector: 'app-admissions',
  standalone: true,
  imports: [CommonModule, ReactiveFormsModule],
  template: `
    <h2 class="section-title">Admission Inquiry</h2>

    @if (submittedInquiry) {
      <div class="alert alert-success">
        Thank you! Your inquiry number is <strong>{{ submittedInquiry.inquiryNumber }}</strong>.
        Our admissions team will contact you soon.
        <div class="mt-2">
          <button class="btn btn-sm btn-outline-success" (click)="submittedInquiry = null">Submit another inquiry</button>
        </div>
      </div>
    } @else {
      <div class="card card-shadow" style="max-width: 720px;">
        <div class="card-body">
          <form [formGroup]="form" (ngSubmit)="submit()">
            <div class="row g-3">
              <div class="col-md-6">
                <label class="form-label">Student Name</label>
                <input type="text" class="form-control" formControlName="studentName">
              </div>
              <div class="col-md-6">
                <label class="form-label">Mobile</label>
                <input type="text" class="form-control" formControlName="mobile">
              </div>
              <div class="col-md-6">
                <label class="form-label">Email</label>
                <input type="email" class="form-control" formControlName="email">
              </div>
              <div class="col-md-6">
                <label class="form-label">Course Interested</label>
                <select class="form-select" formControlName="courseInterest">
                  <option value="" disabled>Select a course</option>
                  @for (course of courses; track course.id) {
                    <option [value]="course.courseCode">{{ course.courseName }}</option>
                  }
                </select>
              </div>
              <div class="col-md-6">
                <label class="form-label">City</label>
                <input type="text" class="form-control" formControlName="city">
              </div>
              <div class="col-md-6">
                <label class="form-label">State</label>
                <input type="text" class="form-control" formControlName="state">
              </div>
              <div class="col-md-6">
                <label class="form-label">Current Qualification</label>
                <input type="text" class="form-control" formControlName="qualification">
              </div>
              <div class="col-12">
                <label class="form-label">Remarks</label>
                <textarea class="form-control" rows="3" formControlName="remarks"></textarea>
              </div>
            </div>
            <button type="submit" class="btn btn-primary mt-3" [disabled]="form.invalid || submitting">
              {{ submitting ? 'Submitting...' : 'Submit Inquiry' }}
            </button>
          </form>
        </div>
      </div>
    }
  `
})
export class AdmissionsComponent implements OnInit {
  courses: Course[] = [];
  submitting = false;
  submittedInquiry: AdmissionInquiry | null = null;

  readonly form = this.fb.group({
    studentName: ['', [Validators.required, Validators.maxLength(255)]],
    mobile: ['', [Validators.required, Validators.pattern(/^[0-9+\-() ]{7,20}$/)]],
    email: ['', [Validators.email]],
    city: ['', [Validators.maxLength(100)]],
    state: ['', [Validators.maxLength(100)]],
    courseInterest: ['', [Validators.required]],
    qualification: ['', [Validators.maxLength(255)]],
    remarks: ['', [Validators.maxLength(2000)]]
  });

  constructor(
    private readonly fb: FormBuilder,
    private readonly admissionService: AdmissionService,
    private readonly courseService: CourseService,
    private readonly notificationService: NotificationService
  ) {}

  ngOnInit(): void {
    this.courseService.getAllCourses(0, 50).subscribe({
      next: response => (this.courses = response.data.content)
    });
  }

  submit(): void {
    if (this.form.invalid) {
      this.form.markAllAsTouched();
      return;
    }

    this.submitting = true;
    this.admissionService.submitInquiry(this.form.getRawValue() as any).subscribe({
      next: response => {
        this.submittedInquiry = response.data;
        this.notificationService.success(response.message);
        this.form.reset();
        this.submitting = false;
      },
      error: () => (this.submitting = false)
    });
  }
}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\features\contact\contact.component.ts
import { CommonModule } from '@angular/common';
import { Component } from '@angular/core';
import { FormBuilder, ReactiveFormsModule, Validators } from '@angular/forms';
import { ContactService } from '../../core/services/contact.service';
import { NotificationService } from '../../core/services/notification.service';

@Component({
  selector: 'app-contact',
  standalone: true,
  imports: [CommonModule, ReactiveFormsModule],
  template: `
    <h2 class="section-title">Contact Us</h2>
    <div class="row g-4">
      <div class="col-md-6">
        <div class="card card-shadow">
          <div class="card-body">
            <h6 class="card-title">Get in Touch</h6>
            <p class="mb-1"><strong>Address:</strong> 123 College Road, Knowledge City, Springfield, 500001</p>
            <p class="mb-1"><strong>Phone:</strong> +91-40-12345678</p>
            <p class="mb-0"><strong>Email:</strong> info&#64;collegeportal.edu</p>
          </div>
        </div>
      </div>
      <div class="col-md-6">
        <div class="card card-shadow">
          <div class="card-body">
            <h6 class="card-title">Send a Message</h6>
            <form [formGroup]="form" (ngSubmit)="submit()">
              <div class="mb-3">
                <label class="form-label">Name</label>
                <input type="text" class="form-control" formControlName="name">
                @if (form.get('name')?.invalid && form.get('name')?.touched) {
                  <div class="text-danger small">Name is required.</div>
                }
              </div>
              <div class="mb-3">
                <label class="form-label">Mobile</label>
                <input type="text" class="form-control" formControlName="mobile">
                @if (form.get('mobile')?.invalid && form.get('mobile')?.touched) {
                  <div class="text-danger small">A valid mobile number is required.</div>
                }
              </div>
              <div class="mb-3">
                <label class="form-label">Email</label>
                <input type="email" class="form-control" formControlName="email">
                @if (form.get('email')?.invalid && form.get('email')?.touched) {
                  <div class="text-danger small">A valid email is required.</div>
                }
              </div>
              <div class="mb-3">
                <label class="form-label">Subject</label>
                <input type="text" class="form-control" formControlName="subject">
              </div>
              <div class="mb-3">
                <label class="form-label">Message</label>
                <textarea class="form-control" rows="4" formControlName="message"></textarea>
              </div>
              <button type="submit" class="btn btn-primary" [disabled]="form.invalid || submitting">
                {{ submitting ? 'Submitting...' : 'Submit' }}
              </button>
            </form>
          </div>
        </div>
      </div>
    </div>
  `
})
export class ContactComponent {
  submitting = false;

  readonly form = this.fb.group({
    name: ['', [Validators.required, Validators.maxLength(255)]],
    mobile: ['', [Validators.required, Validators.pattern(/^[0-9+\-() ]{7,20}$/)]],
    email: ['', [Validators.required, Validators.email]],
    subject: ['', [Validators.required, Validators.maxLength(255)]],
    message: ['', [Validators.required, Validators.maxLength(4000)]]
  });

  constructor(
    private readonly fb: FormBuilder,
    private readonly contactService: ContactService,
    private readonly notificationService: NotificationService
  ) {}

  submit(): void {
    if (this.form.invalid) {
      this.form.markAllAsTouched();
      return;
    }

    this.submitting = true;
    this.contactService.submitContact(this.form.getRawValue() as any).subscribe({
      next: response => {
        this.notificationService.success(response.message);
        this.form.reset();
        this.submitting = false;
      },
      error: () => (this.submitting = false)
    });
  }
}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\features\courses\courses.component.ts
import { CommonModule } from '@angular/common';
import { Component, OnInit } from '@angular/core';
import { MatPaginatorModule, PageEvent } from '@angular/material/paginator';
import { CourseService } from '../../core/services/course.service';
import { Course } from '../../core/models/course.model';
import { LoadingSpinnerComponent } from '../../shared/components/loading-spinner/loading-spinner.component';

@Component({
  selector: 'app-courses',
  standalone: true,
  imports: [CommonModule, MatPaginatorModule, LoadingSpinnerComponent],
  template: `
    <h2 class="section-title">Our Courses</h2>
    @if (loading) {
      <app-loading-spinner></app-loading-spinner>
    } @else {
      <div class="row g-4">
        @for (course of courses; track course.id) {
          <div class="col-md-4">
            <div class="card card-shadow h-100">
              <div class="card-body">
                <h5 class="card-title">{{ course.courseName }} <span class="badge bg-secondary">{{ course.courseCode }}</span></h5>
                <p class="mb-1"><strong>Duration:</strong> {{ course.duration }}</p>
                <p class="mb-1"><strong>Eligibility:</strong> {{ course.eligibility }}</p>
                <p class="mb-1"><strong>Fees:</strong> &#8377;{{ course.fees | number }}</p>
                <p class="mb-1"><strong>Intake:</strong> {{ course.intakeCapacity }}</p>
                <p class="card-text">{{ course.description }}</p>
                <p class="mb-0"><strong>Career Opportunities:</strong> {{ course.careerOpportunities }}</p>
              </div>
            </div>
          </div>
        } @empty {
          <p class="text-muted">No courses available.</p>
        }
      </div>
      <mat-paginator
        [length]="totalElements"
        [pageSize]="pageSize"
        [pageIndex]="pageIndex"
        [pageSizeOptions]="[3, 6, 9]"
        (page)="onPageChange($event)"
        class="mt-4">
      </mat-paginator>
    }
  `
})
export class CoursesComponent implements OnInit {
  courses: Course[] = [];
  totalElements = 0;
  pageIndex = 0;
  pageSize = 6;
  loading = true;

  constructor(private readonly courseService: CourseService) {}

  ngOnInit(): void {
    this.loadCourses();
  }

  onPageChange(event: PageEvent): void {
    this.pageIndex = event.pageIndex;
    this.pageSize = event.pageSize;
    this.loadCourses();
  }

  private loadCourses(): void {
    this.loading = true;
    this.courseService.getAllCourses(this.pageIndex, this.pageSize).subscribe({
      next: response => {
        this.courses = response.data.content;
        this.totalElements = response.data.totalElements;
        this.loading = false;
      },
      error: () => (this.loading = false)
    });
  }
}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\features\dashboard\contacts\contacts-list.component.ts
import { CommonModule } from '@angular/common';
import { Component, OnInit } from '@angular/core';
import { MatPaginatorModule, PageEvent } from '@angular/material/paginator';
import { ContactService } from '../../../core/services/contact.service';
import { ContactUs } from '../../../core/models/contact.model';
import { LoadingSpinnerComponent } from '../../../shared/components/loading-spinner/loading-spinner.component';

@Component({
  selector: 'app-contacts-list',
  standalone: true,
  imports: [CommonModule, MatPaginatorModule, LoadingSpinnerComponent],
  template: `
    <h3 class="section-title">Contact Requests</h3>
    @if (loading) {
      <app-loading-spinner></app-loading-spinner>
    } @else {
      <div class="table-responsive">
        <table class="table table-striped card-shadow">
          <thead class="table-dark">
            <tr><th>Name</th><th>Email</th><th>Mobile</th><th>Subject</th><th>Message</th><th>Date</th></tr>
          </thead>
          <tbody>
            @for (contact of contacts; track contact.id) {
              <tr>
                <td>{{ contact.name }}</td>
                <td>{{ contact.email }}</td>
                <td>{{ contact.mobile }}</td>
                <td>{{ contact.subject }}</td>
                <td>{{ contact.message }}</td>
                <td>{{ contact.createdDate | date: 'medium' }}</td>
              </tr>
            } @empty {
              <tr><td colspan="6" class="text-center text-muted">No contact requests found.</td></tr>
            }
          </tbody>
        </table>
      </div>
      <mat-paginator
        [length]="totalElements" [pageSize]="pageSize" [pageIndex]="pageIndex"
        [pageSizeOptions]="[10, 25, 50]" (page)="onPageChange($event)">
      </mat-paginator>
    }
  `
})
export class ContactsListComponent implements OnInit {
  contacts: ContactUs[] = [];
  totalElements = 0;
  pageIndex = 0;
  pageSize = 10;
  loading = true;

  constructor(private readonly contactService: ContactService) {}

  ngOnInit(): void {
    this.loadContacts();
  }

  onPageChange(event: PageEvent): void {
    this.pageIndex = event.pageIndex;
    this.pageSize = event.pageSize;
    this.loadContacts();
  }

  private loadContacts(): void {
    this.loading = true;
    this.contactService.getAllContacts(this.pageIndex, this.pageSize).subscribe({
      next: response => {
        this.contacts = response.data.content;
        this.totalElements = response.data.totalElements;
        this.loading = false;
      },
      error: () => (this.loading = false)
    });
  }
}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\features\dashboard\course-management\course-management.component.ts
import { CommonModule } from '@angular/common';
import { Component, OnInit } from '@angular/core';
import { FormBuilder, ReactiveFormsModule, Validators } from '@angular/forms';
import { MatPaginatorModule, PageEvent } from '@angular/material/paginator';
import { CourseService } from '../../../core/services/course.service';
import { Course } from '../../../core/models/course.model';
import { AuthService } from '../../../core/services/auth.service';
import { NotificationService } from '../../../core/services/notification.service';
import { Roles } from '../../../core/constants/roles.constant';
import { LoadingSpinnerComponent } from '../../../shared/components/loading-spinner/loading-spinner.component';

@Component({
  selector: 'app-course-management',
  standalone: true,
  imports: [CommonModule, ReactiveFormsModule, MatPaginatorModule, LoadingSpinnerComponent],
  template: `
    <div class="d-flex justify-content-between align-items-center mb-3">
      <h3 class="section-title mb-0">Course Management</h3>
      @if (canEdit) {
        <button class="btn btn-primary" (click)="openCreateForm()">Add Course</button>
      }
    </div>

    @if (showForm) {
      <div class="card card-shadow mb-4">
        <div class="card-body">
          <h6 class="card-title">{{ editingId ? 'Edit Course' : 'New Course' }}</h6>
          <form [formGroup]="form" (ngSubmit)="save()">
            <div class="row g-3">
              <div class="col-md-4">
                <label class="form-label">Course Code</label>
                <input type="text" class="form-control" formControlName="courseCode">
              </div>
              <div class="col-md-4">
                <label class="form-label">Course Name</label>
                <input type="text" class="form-control" formControlName="courseName">
              </div>
              <div class="col-md-4">
                <label class="form-label">Duration</label>
                <input type="text" class="form-control" formControlName="duration">
              </div>
              <div class="col-md-4">
                <label class="form-label">Fees</label>
                <input type="number" class="form-control" formControlName="fees">
              </div>
              <div class="col-md-4">
                <label class="form-label">Intake Capacity</label>
                <input type="number" class="form-control" formControlName="intakeCapacity">
              </div>
              <div class="col-md-4">
                <label class="form-label">Status</label>
                <select class="form-select" formControlName="status">
                  <option value="ACTIVE">ACTIVE</option>
                  <option value="INACTIVE">INACTIVE</option>
                </select>
              </div>
              <div class="col-md-6">
                <label class="form-label">Eligibility</label>
                <input type="text" class="form-control" formControlName="eligibility">
              </div>
              <div class="col-md-6">
                <label class="form-label">Career Opportunities</label>
                <input type="text" class="form-control" formControlName="careerOpportunities">
              </div>
              <div class="col-12">
                <label class="form-label">Description</label>
                <textarea class="form-control" rows="2" formControlName="description"></textarea>
              </div>
            </div>
            <div class="mt-3">
              <button type="submit" class="btn btn-success me-2" [disabled]="form.invalid">Save</button>
              <button type="button" class="btn btn-secondary" (click)="closeForm()">Cancel</button>
            </div>
          </form>
        </div>
      </div>
    }

    @if (loading) {
      <app-loading-spinner></app-loading-spinner>
    } @else {
      <div class="table-responsive">
        <table class="table table-striped card-shadow">
          <thead class="table-dark">
            <tr>
              <th>Code</th><th>Name</th><th>Duration</th><th>Fees</th><th>Status</th><th>Actions</th>
            </tr>
          </thead>
          <tbody>
            @for (course of courses; track course.id) {
              <tr>
                <td>{{ course.courseCode }}</td>
                <td>{{ course.courseName }}</td>
                <td>{{ course.duration }}</td>
                <td>{{ course.fees | number }}</td>
                <td>{{ course.status }}</td>
                <td>
                  @if (canEdit) {
                    <button class="btn btn-sm btn-outline-primary me-2" (click)="openEditForm(course)">Edit</button>
                  }
                  @if (canDelete) {
                    <button class="btn btn-sm btn-outline-danger" (click)="deleteCourse(course.id)">Delete</button>
                  }
                </td>
              </tr>
            }
          </tbody>
        </table>
      </div>
      <mat-paginator
        [length]="totalElements" [pageSize]="pageSize" [pageIndex]="pageIndex"
        [pageSizeOptions]="[5, 10, 25]" (page)="onPageChange($event)">
      </mat-paginator>
    }
  `
})
export class CourseManagementComponent implements OnInit {
  courses: Course[] = [];
  totalElements = 0;
  pageIndex = 0;
  pageSize = 10;
  loading = true;
  showForm = false;
  editingId: number | null = null;

  readonly form = this.fb.group({
    courseCode: ['', Validators.required],
    courseName: ['', Validators.required],
    duration: [''],
    eligibility: [''],
    fees: [0, [Validators.required, Validators.min(0)]],
    description: [''],
    intakeCapacity: [1, [Validators.required, Validators.min(1)]],
    careerOpportunities: [''],
    status: ['ACTIVE', Validators.required]
  });

  constructor(
    private readonly fb: FormBuilder,
    private readonly courseService: CourseService,
    private readonly authService: AuthService,
    private readonly notificationService: NotificationService
  ) {}

  get canEdit(): boolean {
    return this.authService.hasAnyRole([Roles.ADMIN, Roles.EDITOR]);
  }

  get canDelete(): boolean {
    return this.authService.hasAnyRole([Roles.ADMIN]);
  }

  ngOnInit(): void {
    this.loadCourses();
  }

  onPageChange(event: PageEvent): void {
    this.pageIndex = event.pageIndex;
    this.pageSize = event.pageSize;
    this.loadCourses();
  }

  openCreateForm(): void {
    this.editingId = null;
    this.form.reset({ status: 'ACTIVE', fees: 0, intakeCapacity: 1 });
    this.showForm = true;
  }

  openEditForm(course: Course): void {
    this.editingId = course.id;
    this.form.patchValue(course);
    this.showForm = true;
  }

  closeForm(): void {
    this.showForm = false;
  }

  save(): void {
    if (this.form.invalid) {
      return;
    }
    const request = this.form.getRawValue() as any;
    const action$ = this.editingId
      ? this.courseService.updateCourse(this.editingId, request)
      : this.courseService.createCourse(request);

    action$.subscribe({
      next: response => {
        this.notificationService.success(response.message);
        this.showForm = false;
        this.loadCourses();
      },
      error: () => undefined
    });
  }

  deleteCourse(id: number): void {
    if (!confirm('Are you sure you want to delete this course?')) {
      return;
    }
    this.courseService.deleteCourse(id).subscribe({
      next: () => {
        this.notificationService.success('Course deleted successfully');
        this.loadCourses();
      },
      error: () => undefined
    });
  }

  private loadCourses(): void {
    this.loading = true;
    this.courseService.getAllCourses(this.pageIndex, this.pageSize).subscribe({
      next: response => {
        this.courses = response.data.content;
        this.totalElements = response.data.totalElements;
        this.loading = false;
      },
      error: () => (this.loading = false)
    });
  }
}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\features\dashboard\faculty-management\faculty-management.component.ts
import { CommonModule } from '@angular/common';
import { Component, OnInit } from '@angular/core';
import { FormBuilder, ReactiveFormsModule, Validators } from '@angular/forms';
import { MatPaginatorModule, PageEvent } from '@angular/material/paginator';
import { FacultyService } from '../../../core/services/faculty.service';
import { Faculty } from '../../../core/models/faculty.model';
import { AuthService } from '../../../core/services/auth.service';
import { NotificationService } from '../../../core/services/notification.service';
import { Roles } from '../../../core/constants/roles.constant';
import { LoadingSpinnerComponent } from '../../../shared/components/loading-spinner/loading-spinner.component';

@Component({
  selector: 'app-faculty-management',
  standalone: true,
  imports: [CommonModule, ReactiveFormsModule, MatPaginatorModule, LoadingSpinnerComponent],
  template: `
    <div class="d-flex justify-content-between align-items-center mb-3">
      <h3 class="section-title mb-0">Faculty Management</h3>
      @if (canEdit) {
        <button class="btn btn-primary" (click)="openCreateForm()">Add Faculty</button>
      }
    </div>

    @if (showForm) {
      <div class="card card-shadow mb-4">
        <div class="card-body">
          <h6 class="card-title">{{ editingId ? 'Edit Faculty' : 'New Faculty' }}</h6>
          <form [formGroup]="form" (ngSubmit)="save()">
            <div class="row g-3">
              <div class="col-md-4">
                <label class="form-label">Name</label>
                <input type="text" class="form-control" formControlName="name">
              </div>
              <div class="col-md-4">
                <label class="form-label">Department</label>
                <input type="text" class="form-control" formControlName="department">
              </div>
              <div class="col-md-4">
                <label class="form-label">Designation</label>
                <input type="text" class="form-control" formControlName="designation">
              </div>
              <div class="col-md-4">
                <label class="form-label">Qualification</label>
                <input type="text" class="form-control" formControlName="qualification">
              </div>
              <div class="col-md-4">
                <label class="form-label">Experience (years)</label>
                <input type="number" class="form-control" formControlName="experience">
              </div>
              <div class="col-md-4">
                <label class="form-label">Photo URL</label>
                <input type="text" class="form-control" formControlName="photoUrl">
              </div>
              <div class="col-12">
                <label class="form-label">Specialization</label>
                <input type="text" class="form-control" formControlName="specialization">
              </div>
            </div>
            <div class="mt-3">
              <button type="submit" class="btn btn-success me-2" [disabled]="form.invalid">Save</button>
              <button type="button" class="btn btn-secondary" (click)="closeForm()">Cancel</button>
            </div>
          </form>
        </div>
      </div>
    }

    @if (loading) {
      <app-loading-spinner></app-loading-spinner>
    } @else {
      <div class="table-responsive">
        <table class="table table-striped card-shadow">
          <thead class="table-dark">
            <tr>
              <th>Name</th><th>Department</th><th>Designation</th><th>Experience</th><th>Actions</th>
            </tr>
          </thead>
          <tbody>
            @for (faculty of facultyList; track faculty.id) {
              <tr>
                <td>{{ faculty.name }}</td>
                <td>{{ faculty.department }}</td>
                <td>{{ faculty.designation }}</td>
                <td>{{ faculty.experience }}</td>
                <td>
                  @if (canEdit) {
                    <button class="btn btn-sm btn-outline-primary me-2" (click)="openEditForm(faculty)">Edit</button>
                  }
                  @if (canDelete) {
                    <button class="btn btn-sm btn-outline-danger" (click)="deleteFaculty(faculty.id)">Delete</button>
                  }
                </td>
              </tr>
            }
          </tbody>
        </table>
      </div>
      <mat-paginator
        [length]="totalElements" [pageSize]="pageSize" [pageIndex]="pageIndex"
        [pageSizeOptions]="[5, 10, 25]" (page)="onPageChange($event)">
      </mat-paginator>
    }
  `
})
export class FacultyManagementComponent implements OnInit {
  facultyList: Faculty[] = [];
  totalElements = 0;
  pageIndex = 0;
  pageSize = 10;
  loading = true;
  showForm = false;
  editingId: number | null = null;

  readonly form = this.fb.group({
    name: ['', Validators.required],
    department: ['', Validators.required],
    designation: [''],
    qualification: [''],
    experience: [0, [Validators.min(0)]],
    specialization: [''],
    photoUrl: ['']
  });

  constructor(
    private readonly fb: FormBuilder,
    private readonly facultyService: FacultyService,
    private readonly authService: AuthService,
    private readonly notificationService: NotificationService
  ) {}

  get canEdit(): boolean {
    return this.authService.hasAnyRole([Roles.ADMIN, Roles.EDITOR]);
  }

  get canDelete(): boolean {
    return this.authService.hasAnyRole([Roles.ADMIN]);
  }

  ngOnInit(): void {
    this.loadFaculty();
  }

  onPageChange(event: PageEvent): void {
    this.pageIndex = event.pageIndex;
    this.pageSize = event.pageSize;
    this.loadFaculty();
  }

  openCreateForm(): void {
    this.editingId = null;
    this.form.reset({ experience: 0 });
    this.showForm = true;
  }

  openEditForm(faculty: Faculty): void {
    this.editingId = faculty.id;
    this.form.patchValue(faculty);
    this.showForm = true;
  }

  closeForm(): void {
    this.showForm = false;
  }

  save(): void {
    if (this.form.invalid) {
      return;
    }
    const request = this.form.getRawValue() as any;
    const action$ = this.editingId
      ? this.facultyService.updateFaculty(this.editingId, request)
      : this.facultyService.createFaculty(request);

    action$.subscribe({
      next: response => {
        this.notificationService.success(response.message);
        this.showForm = false;
        this.loadFaculty();
      },
      error: () => undefined
    });
  }

  deleteFaculty(id: number): void {
    if (!confirm('Are you sure you want to delete this faculty member?')) {
      return;
    }
    this.facultyService.deleteFaculty(id).subscribe({
      next: () => {
        this.notificationService.success('Faculty deleted successfully');
        this.loadFaculty();
      },
      error: () => undefined
    });
  }

  private loadFaculty(): void {
    this.loading = true;
    this.facultyService.getAllFaculty(this.pageIndex, this.pageSize).subscribe({
      next: response => {
        this.facultyList = response.data.content;
        this.totalElements = response.data.totalElements;
        this.loading = false;
      },
      error: () => (this.loading = false)
    });
  }
}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\features\dashboard\inquiry-management\inquiry-management.component.ts
import { CommonModule } from '@angular/common';
import { Component, OnInit } from '@angular/core';
import { FormsModule } from '@angular/forms';
import { MatPaginatorModule, PageEvent } from '@angular/material/paginator';
import { AdmissionService } from '../../../core/services/admission.service';
import { AdmissionInquiry, InquiryStatus } from '../../../core/models/admission-inquiry.model';
import { AuthService } from '../../../core/services/auth.service';
import { NotificationService } from '../../../core/services/notification.service';
import { Roles } from '../../../core/constants/roles.constant';
import { LoadingSpinnerComponent } from '../../../shared/components/loading-spinner/loading-spinner.component';

const STATUSES: InquiryStatus[] = ['NEW', 'CONTACTED', 'INTERESTED', 'VISITED', 'ADMITTED', 'CLOSED'];

@Component({
  selector: 'app-inquiry-management',
  standalone: true,
  imports: [CommonModule, FormsModule, MatPaginatorModule, LoadingSpinnerComponent],
  template: `
    <div class="d-flex justify-content-between align-items-center mb-3 flex-wrap gap-2">
      <h3 class="section-title mb-0">Admission Inquiry Management</h3>
      @if (canManage) {
        <a class="btn btn-outline-secondary" [href]="exportUrl" target="_blank">Export CSV</a>
      }
    </div>

    <div class="row g-2 mb-3">
      <div class="col-md-4">
        <input type="text" class="form-control" placeholder="Search by name, mobile or inquiry number"
               [(ngModel)]="keyword" (keyup.enter)="search()">
      </div>
      <div class="col-md-3">
        <select class="form-select" [(ngModel)]="statusFilter" (change)="search()">
          <option value="">All Statuses</option>
          @for (status of statuses; track status) {
            <option [value]="status">{{ status }}</option>
          }
        </select>
      </div>
      <div class="col-md-2">
        <button class="btn btn-primary" (click)="search()">Filter</button>
      </div>
    </div>

    @if (loading) {
      <app-loading-spinner></app-loading-spinner>
    } @else {
      <div class="table-responsive">
        <table class="table table-striped card-shadow">
          <thead class="table-dark">
            <tr>
              <th>Inquiry #</th><th>Student</th><th>Mobile</th><th>Course</th><th>Status</th><th>Remarks</th><th>Actions</th>
            </tr>
          </thead>
          <tbody>
            @for (inquiry of inquiries; track inquiry.id) {
              <tr>
                <td>{{ inquiry.inquiryNumber }}</td>
                <td>{{ inquiry.studentName }}</td>
                <td>{{ inquiry.mobile }}</td>
                <td>{{ inquiry.courseInterest }}</td>
                <td><span class="badge bg-info text-dark">{{ inquiry.status }}</span></td>
                <td>{{ inquiry.remarks }}</td>
                <td>
                  @if (canManage) {
                    <button class="btn btn-sm btn-outline-primary" (click)="openStatusEditor(inquiry)">Update</button>
                  }
                </td>
              </tr>
              @if (editingId === inquiry.id) {
                <tr>
                  <td colspan="7">
                    <div class="d-flex gap-2 align-items-start flex-wrap">
                      <select class="form-select" style="max-width: 200px;" [(ngModel)]="newStatus" [ngModelOptions]="{standalone: true}">
                        @for (status of statuses; track status) {
                          <option [value]="status">{{ status }}</option>
                        }
                      </select>
                      <input type="text" class="form-control" style="max-width: 320px;"
                             placeholder="Remarks" [(ngModel)]="newRemarks" [ngModelOptions]="{standalone: true}">
                      <button class="btn btn-success" (click)="saveStatus(inquiry.id)">Save</button>
                      <button class="btn btn-secondary" (click)="editingId = null">Cancel</button>
                    </div>
                  </td>
                </tr>
              }
            } @empty {
              <tr><td colspan="7" class="text-center text-muted">No inquiries found.</td></tr>
            }
          </tbody>
        </table>
      </div>
      <mat-paginator
        [length]="totalElements" [pageSize]="pageSize" [pageIndex]="pageIndex"
        [pageSizeOptions]="[10, 25, 50]" (page)="onPageChange($event)">
      </mat-paginator>
    }
  `
})
export class InquiryManagementComponent implements OnInit {
  inquiries: AdmissionInquiry[] = [];
  totalElements = 0;
  pageIndex = 0;
  pageSize = 10;
  loading = true;
  keyword = '';
  statusFilter = '';
  statuses = STATUSES;
  editingId: number | null = null;
  newStatus: InquiryStatus = 'NEW';
  newRemarks = '';

  constructor(
    private readonly admissionService: AdmissionService,
    private readonly authService: AuthService,
    private readonly notificationService: NotificationService
  ) {}

  get canManage(): boolean {
    return this.authService.hasAnyRole([Roles.ADMIN, Roles.EDITOR]);
  }

  get exportUrl(): string {
    return this.admissionService.exportCsvUrl();
  }

  ngOnInit(): void {
    this.loadInquiries();
  }

  onPageChange(event: PageEvent): void {
    this.pageIndex = event.pageIndex;
    this.pageSize = event.pageSize;
    this.loadInquiries();
  }

  search(): void {
    this.pageIndex = 0;
    this.loadInquiries();
  }

  openStatusEditor(inquiry: AdmissionInquiry): void {
    this.editingId = inquiry.id;
    this.newStatus = inquiry.status;
    this.newRemarks = inquiry.remarks ?? '';
  }

  saveStatus(id: number): void {
    this.admissionService.updateStatus(id, { status: this.newStatus, remarks: this.newRemarks }).subscribe({
      next: response => {
        this.notificationService.success(response.message);
        this.editingId = null;
        this.loadInquiries();
      },
      error: () => undefined
    });
  }

  private loadInquiries(): void {
    this.loading = true;
    const request$ = this.statusFilter
      ? this.admissionService.getInquiriesByStatus(this.statusFilter, this.pageIndex, this.pageSize)
      : this.keyword.trim()
        ? this.admissionService.searchInquiries(this.keyword.trim(), this.pageIndex, this.pageSize)
        : this.admissionService.getAllInquiries(this.pageIndex, this.pageSize);

    request$.subscribe({
      next: response => {
        this.inquiries = response.data.content;
        this.totalElements = response.data.totalElements;
        this.loading = false;
      },
      error: () => (this.loading = false)
    });
  }
}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\features\dashboard\overview\dashboard-overview.component.ts
import { CommonModule } from '@angular/common';
import { Component, OnInit } from '@angular/core';
import { DashboardService } from '../../../core/services/dashboard.service';
import { DashboardSummary } from '../../../core/models/dashboard.model';
import { LoadingSpinnerComponent } from '../../../shared/components/loading-spinner/loading-spinner.component';

@Component({
  selector: 'app-dashboard-overview',
  standalone: true,
  imports: [CommonModule, LoadingSpinnerComponent],
  template: `
    <h3 class="section-title">Overview</h3>
    @if (loading) {
      <app-loading-spinner></app-loading-spinner>
    } @else if (summary) {
      <div class="row g-3 mb-4">
        <div class="col-md-3">
          <div class="card card-shadow text-center">
            <div class="card-body">
              <h6>Total Courses</h6>
              <p class="fs-3 mb-0">{{ summary.totalCourses }}</p>
            </div>
          </div>
        </div>
        <div class="col-md-3">
          <div class="card card-shadow text-center">
            <div class="card-body">
              <h6>Total Faculty</h6>
              <p class="fs-3 mb-0">{{ summary.totalFaculty }}</p>
            </div>
          </div>
        </div>
        <div class="col-md-3">
          <div class="card card-shadow text-center">
            <div class="card-body">
              <h6>Admission Inquiries</h6>
              <p class="fs-3 mb-0">{{ summary.totalAdmissionInquiries }}</p>
            </div>
          </div>
        </div>
        <div class="col-md-3">
          <div class="card card-shadow text-center">
            <div class="card-body">
              <h6>Contact Requests</h6>
              <p class="fs-3 mb-0">{{ summary.totalContacts }}</p>
            </div>
          </div>
        </div>
      </div>

      <div class="row g-4">
        <div class="col-md-4">
          <div class="card card-shadow h-100">
            <div class="card-body">
              <h6 class="card-title">Course Wise Inquiries</h6>
              @for (point of summary.courseWiseInquiries; track point.label) {
                <div class="mb-2">
                  <div class="d-flex justify-content-between"><span>{{ point.label }}</span><span>{{ point.value }}</span></div>
                  <div class="progress" style="height: 8px;">
                    <div class="progress-bar" [style.width.%]="barWidth(point.value, summary.courseWiseInquiries)"></div>
                  </div>
                </div>
              }
            </div>
          </div>
        </div>
        <div class="col-md-4">
          <div class="card card-shadow h-100">
            <div class="card-body">
              <h6 class="card-title">Monthly Inquiries</h6>
              @for (point of summary.monthlyInquiries; track point.label) {
                <div class="mb-2">
                  <div class="d-flex justify-content-between"><span>{{ point.label }}</span><span>{{ point.value }}</span></div>
                  <div class="progress" style="height: 8px;">
                    <div class="progress-bar bg-success" [style.width.%]="barWidth(point.value, summary.monthlyInquiries)"></div>
                  </div>
                </div>
              }
            </div>
          </div>
        </div>
        <div class="col-md-4">
          <div class="card card-shadow h-100">
            <div class="card-body">
              <h6 class="card-title">Daily Requests</h6>
              @for (point of summary.dailyRequests; track point.label) {
                <div class="mb-2">
                  <div class="d-flex justify-content-between"><span>{{ point.label }}</span><span>{{ point.value }}</span></div>
                  <div class="progress" style="height: 8px;">
                    <div class="progress-bar bg-warning" [style.width.%]="barWidth(point.value, summary.dailyRequests)"></div>
                  </div>
                </div>
              }
            </div>
          </div>
        </div>
      </div>
    }
  `
})
export class DashboardOverviewComponent implements OnInit {
  summary: DashboardSummary | null = null;
  loading = true;

  constructor(private readonly dashboardService: DashboardService) {}

  ngOnInit(): void {
    this.dashboardService.getSummary().subscribe({
      next: response => {
        this.summary = response.data;
        this.loading = false;
      },
      error: () => (this.loading = false)
    });
  }

  barWidth(value: number, points: { value: number }[]): number {
    const max = Math.max(...points.map(p => p.value), 1);
    return (value / max) * 100;
  }
}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\features\faculty\faculty.component.ts
import { CommonModule } from '@angular/common';
import { Component, OnInit } from '@angular/core';
import { FormsModule } from '@angular/forms';
import { MatPaginatorModule, PageEvent } from '@angular/material/paginator';
import { FacultyService } from '../../core/services/faculty.service';
import { Faculty } from '../../core/models/faculty.model';
import { LoadingSpinnerComponent } from '../../shared/components/loading-spinner/loading-spinner.component';

@Component({
  selector: 'app-faculty',
  standalone: true,
  imports: [CommonModule, FormsModule, MatPaginatorModule, LoadingSpinnerComponent],
  template: `
    <h2 class="section-title">Our Faculty</h2>

    <div class="input-group mb-4" style="max-width: 400px;">
      <input type="text" class="form-control" placeholder="Search by name or department"
             [(ngModel)]="keyword" (keyup.enter)="search()">
      <button class="btn btn-primary" (click)="search()">Search</button>
      <button class="btn btn-outline-secondary" (click)="clearSearch()">Clear</button>
    </div>

    @if (loading) {
      <app-loading-spinner></app-loading-spinner>
    } @else {
      <div class="row g-4">
        @for (faculty of facultyList; track faculty.id) {
          <div class="col-md-4">
            <div class="card card-shadow h-100">
              <div class="card-body">
                <h5 class="card-title">{{ faculty.name }}</h5>
                <p class="mb-1"><strong>Department:</strong> {{ faculty.department }}</p>
                <p class="mb-1"><strong>Designation:</strong> {{ faculty.designation }}</p>
                <p class="mb-1"><strong>Qualification:</strong> {{ faculty.qualification }}</p>
                <p class="mb-1"><strong>Experience:</strong> {{ faculty.experience }} years</p>
                <p class="mb-0"><strong>Specialization:</strong> {{ faculty.specialization }}</p>
              </div>
            </div>
          </div>
        } @empty {
          <p class="text-muted">No faculty members found.</p>
        }
      </div>
      <mat-paginator
        [length]="totalElements"
        [pageSize]="pageSize"
        [pageIndex]="pageIndex"
        [pageSizeOptions]="[3, 6, 9]"
        (page)="onPageChange($event)"
        class="mt-4">
      </mat-paginator>
    }
  `
})
export class FacultyComponent implements OnInit {
  facultyList: Faculty[] = [];
  totalElements = 0;
  pageIndex = 0;
  pageSize = 6;
  loading = true;
  keyword = '';

  constructor(private readonly facultyService: FacultyService) {}

  ngOnInit(): void {
    this.loadFaculty();
  }

  search(): void {
    this.pageIndex = 0;
    this.loadFaculty();
  }

  clearSearch(): void {
    this.keyword = '';
    this.pageIndex = 0;
    this.loadFaculty();
  }

  onPageChange(event: PageEvent): void {
    this.pageIndex = event.pageIndex;
    this.pageSize = event.pageSize;
    this.loadFaculty();
  }

  private loadFaculty(): void {
    this.loading = true;
    const request$ = this.keyword.trim()
      ? this.facultyService.searchFaculty(this.keyword.trim(), this.pageIndex, this.pageSize)
      : this.facultyService.getAllFaculty(this.pageIndex, this.pageSize);

    request$.subscribe({
      next: response => {
        this.facultyList = response.data.content;
        this.totalElements = response.data.totalElements;
        this.loading = false;
      },
      error: () => (this.loading = false)
    });
  }
}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\features\fees\fees.component.ts
import { CommonModule } from '@angular/common';
import { Component, OnInit } from '@angular/core';
import { FeeService } from '../../core/services/fee.service';
import { FeeStructure } from '../../core/models/fee.model';
import { LoadingSpinnerComponent } from '../../shared/components/loading-spinner/loading-spinner.component';

@Component({
  selector: 'app-fees',
  standalone: true,
  imports: [CommonModule, LoadingSpinnerComponent],
  template: `
    <h2 class="section-title">Fee Structure</h2>
    @if (loading) {
      <app-loading-spinner></app-loading-spinner>
    } @else if (fees) {
      <div class="table-responsive mb-4">
        <table class="table table-striped table-bordered card-shadow">
          <thead class="table-dark">
            <tr>
              <th>Course Code</th>
              <th>Course Name</th>
              <th>Semester Fees (&#8377;)</th>
            </tr>
          </thead>
          <tbody>
            @for (fee of fees.courseFees; track fee.courseCode) {
              <tr>
                <td>{{ fee.courseCode }}</td>
                <td>{{ fee.courseName }}</td>
                <td>{{ fee.semesterFees | number }}</td>
              </tr>
            }
          </tbody>
        </table>
      </div>

      <div class="row g-4">
        <div class="col-md-4">
          <div class="card card-shadow h-100 text-center">
            <div class="card-body">
              <h6>Admission Fees</h6>
              <p class="fs-4">&#8377;{{ fees.admissionFees | number }}</p>
            </div>
          </div>
        </div>
        <div class="col-md-4">
          <div class="card card-shadow h-100 text-center">
            <div class="card-body">
              <h6>Hostel Fees</h6>
              <p class="fs-4">&#8377;{{ fees.hostelFees | number }}</p>
            </div>
          </div>
        </div>
        <div class="col-md-4">
          <div class="card card-shadow h-100 text-center">
            <div class="card-body">
              <h6>Transport Fees</h6>
              <p class="fs-4">&#8377;{{ fees.transportFees | number }}</p>
            </div>
          </div>
        </div>
      </div>

      <div class="card card-shadow mt-4">
        <div class="card-body">
          <h6 class="card-title">Scholarship Information</h6>
          <p class="card-text mb-0">{{ fees.scholarshipInformation }}</p>
        </div>
      </div>
    }
  `
})
export class FeesComponent implements OnInit {
  fees: FeeStructure | null = null;
  loading = true;

  constructor(private readonly feeService: FeeService) {}

  ngOnInit(): void {
    this.feeService.getFeeStructure().subscribe({
      next: response => {
        this.fees = response.data;
        this.loading = false;
      },
      error: () => (this.loading = false)
    });
  }
}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\features\home\home.component.spec.ts
import { TestBed } from '@angular/core/testing';
import { HttpClientTestingModule, HttpTestingController } from '@angular/common/http/testing';
import { provideRouter } from '@angular/router';
import { HomeComponent } from './home.component';
import { environment } from '../../../environments/environment';
import { ApiResponse } from '../../core/models/api-response.model';
import { HomeResponse } from '../../core/models/home.model';

describe('HomeComponent', () => {
  let httpMock: HttpTestingController;

  beforeEach(async () => {
    await TestBed.configureTestingModule({
      imports: [HomeComponent, HttpClientTestingModule],
      providers: [provideRouter([])]
    }).compileComponents();

    httpMock = TestBed.inject(HttpTestingController);
  });

  afterEach(() => httpMock.verify());

  it('should create and load home data', () => {
    const fixture = TestBed.createComponent(HomeComponent);
    fixture.detectChanges();

    const mockResponse: ApiResponse<HomeResponse> = {
      success: true,
      message: 'ok',
      timestamp: new Date().toISOString(),
      data: {
        collegeName: 'Test College',
        logoUrl: '',
        vision: 'Vision',
        mission: 'Mission',
        principalMessage: 'Welcome',
        latestNews: [],
        upcomingEvents: [],
        collegeHighlights: [],
        placementHighlights: [],
        admissionBannerText: 'Apply now'
      }
    };

    const req = httpMock.expectOne(`${environment.apiUrl}/home`);
    req.flush(mockResponse);
    fixture.detectChanges();

    expect(fixture.componentInstance.home?.collegeName).toBe('Test College');
    expect(fixture.componentInstance.loading).toBeFalse();
  });
});

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\features\home\home.component.ts
import { CommonModule } from '@angular/common';
import { Component, OnInit } from '@angular/core';
import { RouterLink } from '@angular/router';
import { HomeService } from '../../core/services/home.service';
import { HomeResponse } from '../../core/models/home.model';
import { LoadingSpinnerComponent } from '../../shared/components/loading-spinner/loading-spinner.component';

@Component({
  selector: 'app-home',
  standalone: true,
  imports: [CommonModule, RouterLink, LoadingSpinnerComponent],
  template: `
    @if (loading) {
      <app-loading-spinner></app-loading-spinner>
    } @else if (home) {
      <div class="p-4 mb-4 bg-primary text-white rounded-3 text-center">
        <h1 class="fw-bold">{{ home.collegeName }}</h1>
        <p class="lead">{{ home.admissionBannerText }}</p>
        <a routerLink="/admissions" class="btn btn-light btn-lg mt-2">Apply for Admission</a>
      </div>

      <div class="row g-4 mb-4">
        <div class="col-md-6">
          <div class="card card-shadow h-100">
            <div class="card-body">
              <h5 class="card-title">Our Vision</h5>
              <p class="card-text">{{ home.vision }}</p>
            </div>
          </div>
        </div>
        <div class="col-md-6">
          <div class="card card-shadow h-100">
            <div class="card-body">
              <h5 class="card-title">Our Mission</h5>
              <p class="card-text">{{ home.mission }}</p>
            </div>
          </div>
        </div>
      </div>

      <div class="card card-shadow mb-4">
        <div class="card-body">
          <h5 class="card-title">Principal's Message</h5>
          <p class="card-text">{{ home.principalMessage }}</p>
        </div>
      </div>

      <div class="row g-4">
        <div class="col-md-3">
          <h6 class="section-title">Latest News</h6>
          <ul class="list-group">
            @for (item of home.latestNews; track item) {
              <li class="list-group-item">{{ item }}</li>
            }
          </ul>
        </div>
        <div class="col-md-3">
          <h6 class="section-title">Upcoming Events</h6>
          <ul class="list-group">
            @for (item of home.upcomingEvents; track item) {
              <li class="list-group-item">{{ item }}</li>
            }
          </ul>
        </div>
        <div class="col-md-3">
          <h6 class="section-title">College Highlights</h6>
          <ul class="list-group">
            @for (item of home.collegeHighlights; track item) {
              <li class="list-group-item">{{ item }}</li>
            }
          </ul>
        </div>
        <div class="col-md-3">
          <h6 class="section-title">Placement Highlights</h6>
          <ul class="list-group">
            @for (item of home.placementHighlights; track item) {
              <li class="list-group-item">{{ item }}</li>
            }
          </ul>
        </div>
      </div>
    }
  `
})
export class HomeComponent implements OnInit {
  home: HomeResponse | null = null;
  loading = true;

  constructor(private readonly homeService: HomeService) {}

  ngOnInit(): void {
    this.homeService.getHome().subscribe({
      next: response => {
        this.home = response.data;
        this.loading = false;
      },
      error: () => (this.loading = false)
    });
  }
}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\features\login\login.component.ts
import { CommonModule } from '@angular/common';
import { Component } from '@angular/core';
import { FormBuilder, ReactiveFormsModule, Validators } from '@angular/forms';
import { ActivatedRoute, Router, RouterLink } from '@angular/router';
import { AuthService } from '../../core/services/auth.service';
import { NotificationService } from '../../core/services/notification.service';

@Component({
  selector: 'app-login',
  standalone: true,
  imports: [CommonModule, ReactiveFormsModule, RouterLink],
  template: `
    <div class="d-flex justify-content-center">
      <div class="card card-shadow mt-5" style="max-width: 420px; width: 100%;">
        <div class="card-body">
          <h4 class="card-title mb-4 text-center">Admin / Staff Login</h4>

          @if (!showForgotPassword) {
            <form [formGroup]="loginForm" (ngSubmit)="login()">
              <div class="mb-3">
                <label class="form-label">Username</label>
                <input type="text" class="form-control" formControlName="username">
              </div>
              <div class="mb-3">
                <label class="form-label">Password</label>
                <input type="password" class="form-control" formControlName="password">
              </div>
              <button type="submit" class="btn btn-primary w-100" [disabled]="loginForm.invalid || loading">
                {{ loading ? 'Signing in...' : 'Login' }}
              </button>
              <div class="text-center mt-3">
                <a role="button" (click)="showForgotPassword = true">Forgot password?</a>
              </div>
            </form>
          } @else {
            <form [formGroup]="forgotForm" (ngSubmit)="forgotPassword()">
              <div class="mb-3">
                <label class="form-label">Username</label>
                <input type="text" class="form-control" formControlName="usernameOrEmail">
              </div>
              <button type="submit" class="btn btn-primary w-100" [disabled]="forgotForm.invalid || loading">
                Send Reset Instructions
              </button>
              <div class="text-center mt-3">
                <a role="button" (click)="showForgotPassword = false">Back to login</a>
              </div>
            </form>
          }
        </div>
      </div>
    </div>
  `
})
export class LoginComponent {
  showForgotPassword = false;
  loading = false;

  readonly loginForm = this.fb.group({
    username: ['', Validators.required],
    password: ['', Validators.required]
  });

  readonly forgotForm = this.fb.group({
    usernameOrEmail: ['', Validators.required]
  });

  constructor(
    private readonly fb: FormBuilder,
    private readonly authService: AuthService,
    private readonly notificationService: NotificationService,
    private readonly router: Router,
    private readonly route: ActivatedRoute
  ) {}

  login(): void {
    if (this.loginForm.invalid) {
      return;
    }
    this.loading = true;
    this.authService.login(this.loginForm.getRawValue() as any).subscribe({
      next: response => {
        this.notificationService.success(response.message);
        const returnUrl = this.route.snapshot.queryParamMap.get('returnUrl') ?? '/dashboard';
        this.router.navigateByUrl(returnUrl);
        this.loading = false;
      },
      error: () => (this.loading = false)
    });
  }

  forgotPassword(): void {
    if (this.forgotForm.invalid) {
      return;
    }
    this.loading = true;
    this.authService.forgotPassword(this.forgotForm.getRawValue() as any).subscribe({
      next: response => {
        this.notificationService.success(response.message);
        this.showForgotPassword = false;
        this.loading = false;
      },
      error: () => (this.loading = false)
    });
  }
}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\shared\components\footer\footer.component.ts
import { Component } from '@angular/core';

@Component({
  selector: 'app-footer',
  standalone: true,
  template: `
    <footer class="bg-dark text-light py-4 mt-auto">
      <div class="container d-flex flex-column flex-md-row justify-content-between align-items-center">
        <p class="mb-2 mb-md-0">&copy; {{ currentYear }} College Digital Foundation Portal. All rights reserved.</p>
        <p class="mb-0">123 College Road, Knowledge City &nbsp;|&nbsp; info&#64;collegeportal.edu &nbsp;|&nbsp; +91-40-12345678</p>
      </div>
    </footer>
  `
})
export class FooterComponent {
  readonly currentYear = new Date().getFullYear();
}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\shared\components\header\header.component.ts
import { CommonModule } from '@angular/common';
import { Component } from '@angular/core';
import { Router, RouterLink, RouterLinkActive } from '@angular/router';
import { AsyncPipe } from '@angular/common';
import { AuthService } from '../../../core/services/auth.service';

@Component({
  selector: 'app-header',
  standalone: true,
  imports: [CommonModule, RouterLink, RouterLinkActive, AsyncPipe],
  template: `
    <nav class="navbar navbar-expand-lg navbar-dark bg-dark">
      <div class="container">
        <a class="navbar-brand fw-bold" routerLink="/home">College Digital Foundation Portal</a>
        <button class="navbar-toggler" type="button" data-bs-toggle="collapse" data-bs-target="#mainNav">
          <span class="navbar-toggler-icon"></span>
        </button>
        <div class="collapse navbar-collapse" id="mainNav">
          <ul class="navbar-nav ms-auto">
            <li class="nav-item"><a class="nav-link" routerLink="/home" routerLinkActive="active">Home</a></li>
            <li class="nav-item"><a class="nav-link" routerLink="/about" routerLinkActive="active">About</a></li>
            <li class="nav-item"><a class="nav-link" routerLink="/courses" routerLinkActive="active">Courses</a></li>
            <li class="nav-item"><a class="nav-link" routerLink="/fees" routerLinkActive="active">Fees</a></li>
            <li class="nav-item"><a class="nav-link" routerLink="/faculty" routerLinkActive="active">Faculty</a></li>
            <li class="nav-item"><a class="nav-link" routerLink="/contact" routerLinkActive="active">Contact</a></li>
            <li class="nav-item"><a class="nav-link" routerLink="/admissions" routerLinkActive="active">Admissions</a></li>
            @if ((authService.currentUser$ | async) === null) {
              <li class="nav-item"><a class="nav-link" routerLink="/login" routerLinkActive="active">Login</a></li>
            } @else {
              <li class="nav-item"><a class="nav-link" routerLink="/dashboard" routerLinkActive="active">Dashboard</a></li>
              <li class="nav-item"><a class="nav-link" role="button" (click)="logout()">Logout</a></li>
            }
          </ul>
        </div>
      </div>
    </nav>
  `
})
export class HeaderComponent {
  constructor(readonly authService: AuthService, private readonly router: Router) {}

  logout(): void {
    this.authService.logout();
    this.router.navigate(['/home']);
  }
}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\shared\components\loading-spinner\loading-spinner.component.ts
import { Component, Input } from '@angular/core';
import { MatProgressSpinnerModule } from '@angular/material/progress-spinner';

@Component({
  selector: 'app-loading-spinner',
  standalone: true,
  imports: [MatProgressSpinnerModule],
  template: `
    <div class="d-flex justify-content-center align-items-center py-5">
      <mat-spinner [diameter]="diameter"></mat-spinner>
    </div>
  `
})
export class LoadingSpinnerComponent {
  @Input() diameter = 48;
}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\shared\layouts\admin-layout\admin-layout.component.ts
import { CommonModule } from '@angular/common';
import { Component } from '@angular/core';
import { Router, RouterLink, RouterLinkActive, RouterOutlet } from '@angular/router';
import { MatIconModule } from '@angular/material/icon';
import { MatListModule } from '@angular/material/list';
import { MatSidenavModule } from '@angular/material/sidenav';
import { MatToolbarModule } from '@angular/material/toolbar';
import { MatButtonModule } from '@angular/material/button';
import { AuthService } from '../../../core/services/auth.service';

@Component({
  selector: 'app-admin-layout',
  standalone: true,
  imports: [
    CommonModule,
    RouterOutlet,
    RouterLink,
    RouterLinkActive,
    MatSidenavModule,
    MatToolbarModule,
    MatListModule,
    MatIconModule,
    MatButtonModule
  ],
  template: `
    <mat-toolbar color="primary">
      <span>Admin Dashboard</span>
      <span class="flex-spacer"></span>
      <span class="me-3">{{ (authService.currentUser$ | async)?.username }} ({{ (authService.currentUser$ | async)?.role }})</span>
      <button mat-stroked-button color="warn" (click)="logout()">Logout</button>
    </mat-toolbar>
    <mat-sidenav-container class="admin-container">
      <mat-sidenav mode="side" opened class="admin-sidenav">
        <mat-nav-list>
          <a mat-list-item routerLink="/dashboard" routerLinkActive="active-link">Overview</a>
          <a mat-list-item routerLink="/dashboard/courses" routerLinkActive="active-link">Courses</a>
          <a mat-list-item routerLink="/dashboard/faculty" routerLinkActive="active-link">Faculty</a>
          <a mat-list-item routerLink="/dashboard/admissions" routerLinkActive="active-link">Admission Inquiries</a>
          <a mat-list-item routerLink="/dashboard/contacts" routerLinkActive="active-link">Contact Requests</a>
        </mat-nav-list>
      </mat-sidenav>
      <mat-sidenav-content class="p-4">
        <router-outlet></router-outlet>
      </mat-sidenav-content>
    </mat-sidenav-container>
  `,
  styles: [`
    .flex-spacer { flex: 1 1 auto; }
    .admin-container { height: calc(100vh - 64px); }
    .admin-sidenav { width: 240px; }
    .active-link { background-color: rgba(0,0,0,0.06); }
  `]
})
export class AdminLayoutComponent {
  constructor(readonly authService: AuthService, private readonly router: Router) {}

  logout(): void {
    this.authService.logout();
    this.router.navigate(['/home']);
  }
}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\shared\layouts\public-layout\public-layout.component.ts
import { Component } from '@angular/core';
import { RouterOutlet } from '@angular/router';
import { HeaderComponent } from '../../components/header/header.component';
import { FooterComponent } from '../../components/footer/footer.component';

@Component({
  selector: 'app-public-layout',
  standalone: true,
  imports: [RouterOutlet, HeaderComponent, FooterComponent],
  template: `
    <div class="d-flex flex-column min-vh-100">
      <app-header></app-header>
      <main class="page-container flex-grow-1">
        <div class="container">
          <router-outlet></router-outlet>
        </div>
      </main>
      <app-footer></app-footer>
    </div>
  `
})
export class PublicLayoutComponent {}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\app.component.ts
import { Component } from '@angular/core';
import { RouterOutlet } from '@angular/router';

@Component({
  selector: 'app-root',
  standalone: true,
  imports: [RouterOutlet],
  template: '<router-outlet></router-outlet>'
})
export class AppComponent {}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\app.config.ts
import { ApplicationConfig } from '@angular/core';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { provideAnimations } from '@angular/platform-browser/animations';
import { provideRouter } from '@angular/router';
import { routes } from './app.routes';
import { authInterceptor } from './core/interceptors/auth.interceptor';
import { errorInterceptor } from './core/interceptors/error.interceptor';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    provideAnimations(),
    provideHttpClient(withInterceptors([authInterceptor, errorInterceptor]))
  ]
};

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\app\app.routes.ts
import { Routes } from '@angular/router';
import { PublicLayoutComponent } from './shared/layouts/public-layout/public-layout.component';
import { AdminLayoutComponent } from './shared/layouts/admin-layout/admin-layout.component';
import { authGuard } from './core/guards/auth.guard';
import { roleGuard } from './core/guards/role.guard';
import { Roles } from './core/constants/roles.constant';

export const routes: Routes = [
  {
    path: '',
    component: PublicLayoutComponent,
    children: [
      { path: '', redirectTo: 'home', pathMatch: 'full' },
      { path: 'home', loadComponent: () => import('./features/home/home.component').then(m => m.HomeComponent) },
      { path: 'about', loadComponent: () => import('./features/about/about.component').then(m => m.AboutComponent) },
      { path: 'courses', loadComponent: () => import('./features/courses/courses.component').then(m => m.CoursesComponent) },
      { path: 'fees', loadComponent: () => import('./features/fees/fees.component').then(m => m.FeesComponent) },
      { path: 'faculty', loadComponent: () => import('./features/faculty/faculty.component').then(m => m.FacultyComponent) },
      { path: 'contact', loadComponent: () => import('./features/contact/contact.component').then(m => m.ContactComponent) },
      { path: 'admissions', loadComponent: () => import('./features/admissions/admissions.component').then(m => m.AdmissionsComponent) },
      { path: 'login', loadComponent: () => import('./features/login/login.component').then(m => m.LoginComponent) }
    ]
  },
  {
    path: 'dashboard',
    component: AdminLayoutComponent,
    canActivate: [authGuard],
    children: [
      {
        path: '',
        canActivate: [roleGuard],
        data: { roles: [Roles.ADMIN, Roles.EDITOR, Roles.VIEWER] },
        loadComponent: () =>
          import('./features/dashboard/overview/dashboard-overview.component').then(m => m.DashboardOverviewComponent)
      },
      {
        path: 'courses',
        canActivate: [roleGuard],
        data: { roles: [Roles.ADMIN, Roles.EDITOR, Roles.VIEWER] },
        loadComponent: () =>
          import('./features/dashboard/course-management/course-management.component').then(m => m.CourseManagementComponent)
      },
      {
        path: 'faculty',
        canActivate: [roleGuard],
        data: { roles: [Roles.ADMIN, Roles.EDITOR, Roles.VIEWER] },
        loadComponent: () =>
          import('./features/dashboard/faculty-management/faculty-management.component').then(m => m.FacultyManagementComponent)
      },
      {
        path: 'admissions',
        canActivate: [roleGuard],
        data: { roles: [Roles.ADMIN, Roles.EDITOR, Roles.VIEWER] },
        loadComponent: () =>
          import('./features/dashboard/inquiry-management/inquiry-management.component').then(m => m.InquiryManagementComponent)
      },
      {
        path: 'contacts',
        canActivate: [roleGuard],
        data: { roles: [Roles.ADMIN, Roles.EDITOR, Roles.VIEWER] },
        loadComponent: () =>
          import('./features/dashboard/contacts/contacts-list.component').then(m => m.ContactsListComponent)
      }
    ]
  },
  { path: '**', redirectTo: 'home' }
];

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\assets\.gitkeep

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\environments\environment.gcp.ts
export const environment = {
  production: true,
  apiUrl: 'https://backend-url/api/v1',
  analyticsEnabled: true
};

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\environments\environment.local.ts
export const environment = {
  production: false,
  apiUrl: 'http://localhost:8080/api/v1',
  analyticsEnabled: false
};

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\environments\environment.ts
export const environment = {
  production: false,
  apiUrl: 'http://localhost:8080/api/v1',
  analyticsEnabled: false
};

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\favicon.ico

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\index.html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>College Digital Foundation Portal</title>
  <base href="/">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <link rel="icon" type="image/x-icon" href="favicon.ico">
</head>
<body>
  <app-root></app-root>
</body>
</html>

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\main.ts
import { bootstrapApplication } from '@angular/platform-browser';
import { appConfig } from './app/app.config';
import { AppComponent } from './app/app.component';

bootstrapApplication(AppComponent, appConfig).catch(err => console.error(err));

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\src\styles.scss
html, body {
  height: 100%;
  margin: 0;
}

body {
  font-family: Roboto, "Helvetica Neue", Arial, sans-serif;
  background-color: #f5f7fa;
}

.page-container {
  min-height: calc(100vh - 160px);
  padding: 24px 0;
}

.section-title {
  font-weight: 600;
  margin-bottom: 1.5rem;
}

.card-shadow {
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.08);
  border: none;
}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\angular.json
{
  "$schema": "./node_modules/@angular/cli/lib/config/schema.json",
  "version": 1,
  "newProjectRoot": "projects",
  "projects": {
    "college-portal-frontend": {
      "projectType": "application",
      "schematics": {
        "@schematics/angular:component": {
          "style": "scss",
          "standalone": true
        }
      },
      "root": "",
      "sourceRoot": "src",
      "prefix": "app",
      "architect": {
        "build": {
          "builder": "@angular-devkit/build-angular:browser",
          "options": {
            "outputPath": "dist/college-portal-frontend",
            "index": "src/index.html",
            "main": "src/main.ts",
            "polyfills": ["zone.js"],
            "tsConfig": "tsconfig.app.json",
            "assets": ["src/favicon.ico", "src/assets"],
            "styles": [
              "node_modules/bootstrap/dist/css/bootstrap.min.css",
              "@angular/material/prebuilt-themes/indigo-pink.css",
              "src/styles.scss"
            ],
            "scripts": ["node_modules/bootstrap/dist/js/bootstrap.bundle.min.js"]
          },
          "configurations": {
            "production": {
              "budgets": [
                { "type": "initial", "maximumWarning": "1mb", "maximumError": "2mb" },
                { "type": "anyComponentStyle", "maximumWarning": "8kb", "maximumError": "16kb" }
              ],
              "outputHashing": "all",
              "fileReplacements": [
                { "replace": "src/environments/environment.ts", "with": "src/environments/environment.gcp.ts" }
              ]
            },
            "local": {
              "fileReplacements": [
                { "replace": "src/environments/environment.ts", "with": "src/environments/environment.local.ts" }
              ]
            },
            "gcp": {
              "outputHashing": "all",
              "fileReplacements": [
                { "replace": "src/environments/environment.ts", "with": "src/environments/environment.gcp.ts" }
              ]
            },
            "development": {
              "optimization": false,
              "extractLicenses": false,
              "sourceMap": true,
              "namedChunks": true
            }
          },
          "defaultConfiguration": "local"
        },
        "serve": {
          "builder": "@angular-devkit/build-angular:dev-server",
          "configurations": {
            "production": { "buildTarget": "college-portal-frontend:build:production" },
            "development": { "buildTarget": "college-portal-frontend:build:development" },
            "local": { "buildTarget": "college-portal-frontend:build:local" }
          },
          "defaultConfiguration": "development"
        },
        "test": {
          "builder": "@angular-devkit/build-angular:karma",
          "options": {
            "polyfills": ["zone.js", "zone.js/testing"],
            "tsConfig": "tsconfig.spec.json",
            "karmaConfig": "karma.conf.js",
            "assets": ["src/favicon.ico", "src/assets"],
            "styles": [
              "node_modules/bootstrap/dist/css/bootstrap.min.css",
              "src/styles.scss"
            ],
            "scripts": []
          }
        }
      }
    }
  }
}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\Dockerfile
# Build stage
FROM node:20-alpine AS build
WORKDIR /workspace
COPY package.json package-lock.json* ./
RUN npm install --no-audit --no-fund
COPY . .
ARG BUILD_CONFIGURATION=local
RUN npx ng build --configuration=${BUILD_CONFIGURATION}

# Runtime stage
FROM nginx:1.27-alpine
COPY --from=build /workspace/dist/college-portal-frontend /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\karma.conf.js
module.exports = function (config) {
  config.set({
    basePath: '',
    frameworks: ['jasmine', '@angular-devkit/build-angular'],
    plugins: [
      require('karma-jasmine'),
      require('karma-chrome-launcher'),
      require('karma-jasmine-html-reporter'),
      require('karma-coverage'),
      require('@angular-devkit/build-angular/plugins/karma')
    ],
    client: {
      jasmine: {},
      clearContext: false
    },
    jasmineHtmlReporter: {
      suppressAll: true
    },
    coverageReporter: {
      dir: require('path').join(__dirname, './coverage/college-portal-frontend'),
      subdir: '.',
      reporters: [{ type: 'html' }, { type: 'text-summary' }]
    },
    reporters: ['progress', 'kjhtml'],
    port: 9876,
    colors: true,
    logLevel: config.LOG_INFO,
    autoWatch: true,
    customLaunchers: {
      ChromeHeadlessCI: {
        base: 'ChromeHeadless',
        flags: ['--no-sandbox', '--disable-gpu', '--disable-dev-shm-usage']
      }
    },
    singleRun: false,
    restartOnFileChange: true
  });
};

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\nginx.conf
server {
    listen       80;
    server_name  localhost;
    root   /usr/share/nginx/html;
    index  index.html;

    gzip on;
    gzip_types text/plain text/css application/javascript application/json;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location ~* \.(?:css|js|jpg|jpeg|gif|png|svg|ico|woff2?)$ {
        expires 30d;
        add_header Cache-Control "public";
    }
}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\package.json
{
  "name": "college-portal-frontend",
  "version": "1.0.0",
  "scripts": {
    "ng": "ng",
    "start": "ng serve",
    "build": "ng build",
    "build:local": "ng build --configuration=local",
    "build:gcp": "ng build --configuration=gcp",
    "watch": "ng build --watch --configuration development",
    "test": "ng test"
  },
  "private": true,
  "dependencies": {
    "@angular/animations": "17.3.12",
    "@angular/cdk": "17.3.10",
    "@angular/common": "17.3.12",
    "@angular/compiler": "17.3.12",
    "@angular/core": "17.3.12",
    "@angular/forms": "17.3.12",
    "@angular/material": "17.3.10",
    "@angular/platform-browser": "17.3.12",
    "@angular/platform-browser-dynamic": "17.3.12",
    "@angular/router": "17.3.12",
    "bootstrap": "5.3.3",
    "rxjs": "7.8.1",
    "tslib": "2.6.3",
    "zone.js": "0.14.7"
  },
  "devDependencies": {
    "@angular-devkit/build-angular": "17.3.17",
    "@angular/cli": "17.3.17",
    "@angular/compiler-cli": "17.3.12",
    "@types/jasmine": "5.1.4",
    "jasmine-core": "5.1.2",
    "karma": "6.4.4",
    "karma-chrome-launcher": "3.2.0",
    "karma-coverage": "2.2.1",
    "karma-jasmine": "5.1.0",
    "karma-jasmine-html-reporter": "2.1.0",
    "typescript": "5.4.5"
  }
}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\tsconfig.app.json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "outDir": "./out-tsc/app",
    "types": []
  },
  "files": ["src/main.ts"],
  "include": ["src/**/*.d.ts"]
}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\tsconfig.json
{
  "compileOnSave": false,
  "compilerOptions": {
    "outDir": "./dist/out-tsc",
    "forceConsistentCasingInFileNames": true,
    "strict": true,
    "noImplicitOverride": true,
    "noPropertyAccessFromIndexSignature": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "sourceMap": true,
    "declaration": false,
    "experimentalDecorators": true,
    "moduleResolution": "bundler",
    "importHelpers": true,
    "target": "ES2022",
    "module": "ES2022",
    "useDefineForClassFields": false,
    "lib": ["ES2022", "dom"]
  },
  "angularCompilerOptions": {
    "enableI18nLegacyMessageIdFormat": false,
    "strictInjectionParameters": true,
    "strictInputAccessModifiers": true,
    "strictTemplates": true
  }
}

-------------------------------------------------------------
C:\ws\agent\website-college\frontend\tsconfig.spec.json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "outDir": "./out-tsc/spec",
    "types": ["jasmine"]
  },
  "include": ["src/**/*.spec.ts", "src/**/*.d.ts"]
}

-------------------------------------------------------------
C:\ws\agent\website-college\docker-compose.yml
version: "3.9"

services:
  backend:
    build:
      context: ./backend-python
      dockerfile: Dockerfile
    image: college-portal-backend:local
    container_name: college-portal-backend
    ports:
      - "8080:8080"
    environment:
      APP_PROFILE: local
      JWT_SECRET: ${JWT_SECRET:-local-dev-only-change-me-must-be-32-bytes-minimum-secret}
      CORS_ALLOWED_ORIGINS: http://localhost:4200
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:8080/actuator/health')"]
      interval: 15s
      timeout: 5s
      retries: 5

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
      args:
        BUILD_CONFIGURATION: local
    image: college-portal-frontend:local
    container_name: college-portal-frontend
    ports:
      - "4200:80"
    depends_on:
      - backend

-------------------------------------------------------------
C:\ws\agent\website-college\README.md
# College Digital Foundation Portal

Enterprise-grade full-stack portal for digitizing core college operations
(information, courses, fees, faculty, contact, admission inquiries) with an
authenticated admin dashboard. Built as the foundation for future AI-powered
agents (marketing, admissions, helpdesk, placement, parent, fee collection).

## Tech Stack

| Layer    | Technology |
|----------|------------|
| Backend  | Python 3.12, FastAPI, SQLAlchemy, PyJWT, bcrypt, in-memory SQLite |
| Frontend | Angular 17.3, TypeScript 5.4, Angular Material, Bootstrap 5, RxJS |
| Database | In-memory SQLite (seeded at startup), migratable to Cloud SQL PostgreSQL without code changes |
| Deployment | Docker, Docker Compose, Google Cloud Run |

> **Backend rewrite note:** the backend was rewritten from Java/Spring Boot to
> Python/FastAPI (`backend-python/`). The REST API contract (routes, request/
> response envelopes, JWT auth, RBAC) is unchanged, so the Angular frontend
> works as-is with no modifications. The original Java/Spring Boot backend has
> been removed from this repository; Python/FastAPI is now the only backend.

> **Node version note:** Angular CLI 17.3.17 officially supports Node
> `^18.13.0 || ^20.9.0`. Newer Node 26.x has been used successfully for local
> `npm install` / `ng serve` (flagged "Unsupported" by the CLI, but works) —
> Docker images still pin `node:20-alpine` for a guaranteed-supported build
> environment.

## Project Structure

```
college-digital-foundation (this repo)
├── backend-python/      FastAPI REST API (the backend)
├── frontend/            Angular 17 SPA
├── docs/                 Architecture, API, database, deployment, feature-flag docs
├── scripts/              Local dev/ops utility scripts (empty placeholder today)
├── test-data/            Additional sample/fixture data for manual QA
├── docker-compose.yml    Local multi-container orchestration
└── .github/workflows/    CI/CD pipeline
```

See [frontend/README structure](#frontend) below.

## Prerequisites

- Python 3.12+ and pip
- Node.js 18.13+ / 20.9+ and npm 10+ (Angular CLI 17.3.17)
- Docker & Docker Compose (optional, for containerized runs)

## Running Locally (without Docker)

### Backend

```bash
cd backend-python
pip install -r requirements.txt
uvicorn app.main:app --reload --port 8080
```

The backend starts on `http://localhost:8080` with profile `local` (default).
Swagger UI: `http://localhost:8080/swagger-ui.html`

Database: in-memory SQLite, schema created and sample data seeded automatically
on startup (equivalent to the original schema.sql/data.sql/DataInitializer).

Seeded users (BCrypt-encoded at startup, never committed as plain text/hashes):

| Username | Password  | Role         |
|----------|-----------|--------------|
| admin    | admin123  | ROLE_ADMIN   |
| editor   | editor123 | ROLE_EDITOR  |
| viewer   | viewer123 | ROLE_VIEWER  |

### Frontend

```bash
cd frontend
npm install
npm start   # ng serve (development configuration)
```

The app starts on `http://localhost:4200` and calls the backend directly at
`http://localhost:8080/api/v1` (no dev-server proxy — CORS is enabled
backend-side for `http://localhost:4200`). Use `npm run build:local` /
`build:gcp` for environment-specific production builds.

## Running with Docker Compose

```bash
docker-compose up --build
```

- Backend: `http://localhost:8080`
- Frontend: `http://localhost:4200`

## Running Tests

```bash
# Frontend (component, service, guard tests)
cd frontend && npx ng test --watch=false --browsers=ChromeHeadlessCI
```

## Building for Production

```bash
# Backend (build the Docker image)
cd backend-python && docker build -t college-portal-backend:prod .

# Frontend
cd frontend && npx ng build --configuration=gcp
```

## Deployment

See [docs/DEPLOYMENT_GUIDE.md](docs/DEPLOYMENT_GUIDE.md) for Docker and
Google Cloud Run deployment instructions.

## Documentation

- [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)
- [docs/API_DOCUMENTATION.md](docs/API_DOCUMENTATION.md)
- [docs/DATABASE_DESIGN.md](docs/DATABASE_DESIGN.md)
- [docs/DEPLOYMENT_GUIDE.md](docs/DEPLOYMENT_GUIDE.md)
- [docs/FEATURE_FLAGS.md](docs/FEATURE_FLAGS.md)

## License

See [LICENSE](LICENSE).

-------------------------------------------------------------


