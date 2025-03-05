# Online-Training-Database-System
# Import necessary libraries
import streamlit as st
from sqlalchemy import create_engine, Column, Integer, String, ForeignKey, Text, Date
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker
import bcrypt

# Database setup
Base = declarative_base()
engine = create_engine('sqlite:///elearning.db', echo=True)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

# Database Models
class User(Base):
    __tablename__ = 'users'
    id = Column(Integer, primary_key=True, index=True)
    name = Column(String, index=True)
    email = Column(String, unique=True, index=True)
    hashed_password = Column(String)

class Course(Base):
    __tablename__ = 'courses'
    id = Column(Integer, primary_key=True, index=True)
    title = Column(String, index=True)
    description = Column(Text)
    instructor = Column(String)

class Module(Base):
    __tablename__ = 'modules'
    id = Column(Integer, primary_key=True, index=True)
    course_id = Column(Integer, ForeignKey('courses.id'))
    title = Column(String, index=True)
    description = Column(Text)

class Enrolment(Base):
    __tablename__ = 'enrolments'
    id = Column(Integer, primary_key=True, index=True)
    user_id = Column(Integer, ForeignKey('users.id'))
    course_id = Column(Integer, ForeignKey('courses.id'))
    enroll_date = Column(Date)

class Lesson(Base):
    __tablename__ = 'lessons'
    id = Column(Integer, primary_key=True, index=True)
    module_id = Column(Integer, ForeignKey('modules.id'))
    title = Column(String, index=True)
    content = Column(Text)

class Quiz(Base):
    __tablename__ = 'quizzes'
    id = Column(Integer, primary_key=True, index=True)
    lesson_id = Column(Integer, ForeignKey('lessons.id'))
    questions = Column(Text)
    answers = Column(Text)

# Create the database tables
Base.metadata.create_all(bind=engine)

# Utility Functions
def get_user_by_email(db: SessionLocal, email: str):
    return db.query(User).filter(User.email == email).first()

def hash_password(password: str):
    return bcrypt.hashpw(password.encode('utf-8'), bcrypt.gensalt())

def verify_password(plain_password: str, hashed_password: str):
    return bcrypt.checkpw(plain_password.encode('utf-8'), hashed_password)

# Authentication Pages
def signup():
    st.title("Sign Up")
    name = st.text_input("Name")
    email = st.text_input("Email")
    password = st.text_input("Password", type="password")
    if st.button("Sign Up"):
        db = SessionLocal()
        user = get_user_by_email(db, email)
        if user:
            st.error("User already exists")
        else:
            hashed_pw = hash_password(password)
            new_user = User(name=name, email=email, hashed_password=hashed_pw)
            db.add(new_user)
            db.commit()
            st.success("Account created successfully")

def login():
    st.title("Login")
    email = st.text_input("Email")
    password = st.text_input("Password", type="password")
    if st.button("Login"):
        db = SessionLocal()
        user = get_user_by_email(db, email)
        if not user or not verify_password(password, user.hashed_password):
            st.error("Incorrect email or password")
        else:
            st.success("Logged in successfully")
            st.session_state['user'] = user.name
            st.session_state['user_id'] = user.id

# Course Management Pages
def create_course():
    st.title("Create Course")
    title = st.text_input("Course Title")
    description = st.text_area("Description")
    instructor = st.text_input("Instructor")
    if st.button("Create"):
        db = SessionLocal()
        new_course = Course(title=title, description=description, instructor=instructor)
        db.add(new_course)
        db.commit()
        st.success("Course created successfully")
def list_courses():
    st.title("Courses")
    db = SessionLocal()
    courses = db.query(Course).all()
    for course in courses:
        st.subheader(course.title)
        st.write(course.description)
        st.write(f"Instructor: {course.instructor}")
        # Adding a unique key using the course id
        if st.button(f"Enroll in {course.title}", key=f"enroll_{course.id}"):
            user_id = st.session_state.get('user_id')
            if user_id:
                enrol_course(user_id, course.id)
                st.success(f"Enrolled in {course.title}")
            else:
                st.error("You need to log in first")


def enrol_course(user_id, course_id):
    db = SessionLocal()
    new_enrolment = Enrolment(user_id=user_id, course_id=course_id)
    db.add(new_enrolment)
    db.commit()

# Quiz Management Pages
def create_quiz():
    st.title("Create Quiz")
    lesson_id = st.number_input("Lesson ID", min_value=1, step=1)
    questions = st.text_area("Questions")
    answers = st.text_area("Answers")
    if st.button("Create Quiz"):
        db = SessionLocal()
        new_quiz = Quiz(lesson_id=lesson_id, questions=questions, answers=answers)
        db.add(new_quiz)
        db.commit()
        st.success("Quiz created successfully")

def list_quizzes():
    st.title("Quizzes")
    db = SessionLocal()
    quizzes = db.query(Quiz).all()
    for quiz in quizzes:
        st.subheader(f"Quiz ID: {quiz.id}")
        st.write(f"Lesson ID: {quiz.lesson_id}")
        st.write(f"Questions: {quiz.questions}")
        st.write(f"Answers: {quiz.answers}")

# Main Application Entry Point
def main():
    st.sidebar.title("Navigation")
    option = st.sidebar.selectbox("Select Page", ["Sign Up", "Login", "Create Course", "List Courses", "Create Quiz", "List Quizzes"])
    if option == "Sign Up":
        signup()
    elif option == "Login":
        login()
    elif option == "Create Course":
        if 'user' in st.session_state:
            create_course()
        else:
            st.error("Please log in to create a course.")
    elif option == "List Courses":
        list_courses()
    elif option == "Create Quiz":
        if 'user' in st.session_state:
            create_quiz()
        else:
            st.error("Please log in to create a quiz.")
    elif option == "List Quizzes":
        if 'user' in st.session_state:
            list_quizzes()
        else:
            st.error("Please log in to view quizzes.")

if __name__ == '__main__':
    main()
