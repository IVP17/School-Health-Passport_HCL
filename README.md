import sqlite3
import streamlit as st

# --- PAGE CONFIGURATION ---
st.set_page_config(
    page_title="School Health Passport", page_icon="🏥", layout="wide"
)

# --- DATABASE SETUP & MIGRATION ---
DB_NAME = "school_health.db"


def get_connection():
    return sqlite3.connect(DB_NAME, check_same_thread=False)


def generate_roll_id(roll_number, grade, section):
    """Generates ID: gj_roll no._class.section (e.g., 'gj_14_6d')"""
    return f"gj_{str(roll_number).strip()}_{grade}{section.lower()}"


def init_db():
    conn = get_connection()
    cursor = conn.cursor()

    cursor.execute(
        "SELECT name FROM sqlite_master WHERE type='table' AND name='students'"
    )
    table_exists = cursor.fetchone()

    if table_exists:
        cursor.execute("PRAGMA table_info(students)")
        columns = [col[1] for col in cursor.fetchall()]
        if "roll_number" not in columns or "id_number" not in columns:
            cursor.execute("DROP TABLE IF EXISTS students")
            cursor.execute("DROP TABLE IF EXISTS messages")
            cursor.execute("DROP TABLE IF EXISTS parents")
        if "photo" not in columns:
            try:
                cursor.execute("ALTER TABLE students ADD COLUMN photo BLOB")
            except sqlite3.OperationalError:
                pass

    cursor.execute("""
        CREATE TABLE IF NOT EXISTS students (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            id_number TEXT UNIQUE,
            roll_number INTEGER,
            name TEXT,
            grade INTEGER,
            section TEXT,
            blood_group TEXT,
            allergies TEXT,
            emergency_contact TEXT,
            vaccination_history TEXT,
            medications TEXT,
            photo BLOB
        )
    """)

    cursor.execute("""
        CREATE TABLE IF NOT EXISTS messages (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            sender TEXT,
            grade INTEGER,
            section TEXT,
            target TEXT,
            student_name TEXT,
            message TEXT,
            date TEXT
        )
    """)

    cursor.execute("SELECT COUNT(*) FROM students")
    if cursor.fetchone()[0] == 0:
        initial_students = [
            (1, "Aarav Sharma", 5, "A", "B+", "Peanuts, Dust", "+91 98765 43210", "Fully Vaccinated", "None"),
            (14, "Isha V Prasad", 6, "D", "O+", "Penicillin", "+91 91234 56789", "Fully Vaccinated", "Inhaler as needed"),
            (15, "Noopura Nambiar", 6, "D", "A+", "None", "+91 99887 76655", "Fully Vaccinated", "None"),
            (3, "Ananya Rao", 3, "B", "O-", "Dairy", "+91 95555 44444", "Fully Vaccinated", "Lactase enzyme"),
        ]
        
        for roll, name, grade, section, blood, allergies, emergency, vax, meds in initial_students:
            id_num = generate_roll_id(roll, grade, section)
            cursor.execute(
                """
                    INSERT OR IGNORE INTO students (id_number, roll_number, name, grade, section, blood_group, allergies, emergency_contact, vaccination_history, medications)
                    VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
                """,
                (id_num, roll, name, grade, section, blood, allergies, emergency, vax, meds),
            )

    conn.commit()
    conn.close()


init_db()

# --- DESIGN & STYLING ---
st.markdown(
    """
    <style>
    .stApp {
        background-color: #FFFFFF;
        color: #2C2C2C;
        font-family: sans-serif;
    }
    div.stMarkdown, div.stForm, div.stExpander, div[data-testid="stMetric"] {
        background-color: #F7F4EF;
        padding: 1.2rem;
        border-radius: 10px;
        border: 1px solid #E6E0D8;
        margin-bottom: 1rem;
    }
    .stButton>button {
        background-color: #EFECE6;
        color: #2C2C2C;
        border: 1px solid #D5CEC7;
        border-radius: 6px;
        padding: 0.5rem 1rem;
        font-weight: bold;
        width: 100%;
    }
    .stButton>button:hover {
        background-color: #E2DDD5;
        border-color: #BCB4AB;
    }
    h1, h2, h3 {
        color: #3E3832;
    }
    .emergency-box {
        background-color: #FFECEC;
        border: 2px solid #FF4D4D;
        padding: 1.2rem;
        border-radius: 10px;
        margin-bottom: 1rem;
    }
    </style>
""",
    unsafe_allow_html=True,
)

# --- SESSION STATE INITIALIZATION ---
if "logged_in" not in st.session_state:
    st.session_state.logged_in = False
if "user" not in st.session_state:
    st.session_state.user = None
if "nurse_selected_grade" not in st.session_state:
    st.session_state.nurse_selected_grade = None
if "nurse_selected_section" not in st.session_state:
    st.session_state.nurse_selected_section = None
if "teacher_grade" not in st.session_state:
    st.session_state.teacher_grade = 5
if "teacher_section" not in st.session_state:
    st.session_state.teacher_section = "A"


# --- LOGIN PAGE ---
def login_page():
    st.markdown("<h1 style='text-align: center;'>🏥 School Health Passport</h1>", unsafe_allow_html=True)
    st.markdown("<p style='text-align: center; color: #666;'>School Health & Wellness Portal</p>", unsafe_allow_html=True)

    _, col2, _ = st.columns([1, 2, 1])

    with col2:
        st.markdown("### Sign In Portal")
        with st.expander("ℹ️ Demo Accounts / Hints"):
            st.markdown("""
                **Nurse:** `nurse@school.com` / `password123`  
                **Teacher:** `teacher@school.com` / `password123`  
                **Parent:** Select **Parent** and enter your child's **Student ID** (e.g., `gj_14_6d`) directly to view their health passport.
                """)

        selected_role = st.selectbox("Select Role", ["Nurse", "Teacher", "Parent"], key="login_role_select")

        with st.form("login_form"):
            if selected_role == "Parent":
                student_id_input = st.text_input("Child's Student ID (e.g., gj_14_6d)", value="gj_14_6d")
            else:
                email = st.text_input("Email Address", value="nurse@school.com" if selected_role == "Nurse" else "teacher@school.com")
                password = st.text_input("Password", type="password", value="password123")

            submit = st.form_submit_button("Sign In")

            if submit:
                conn = get_connection()
                if selected_role == "Nurse":
                    st.session_state.user = {"name": "School Nurse", "role": "Nurse", "email": email}
                    st.session_state.logged_in = True
                    st.rerun()
                elif selected_role == "Teacher":
                    st.session_state.user = {"name": "Teacher Portal", "role": "Teacher", "email": email}
                    st.session_state.logged_in = True
                    st.rerun()
                else:
                    student_record = conn.execute("SELECT * FROM students WHERE id_number = ?", (student_id_input.strip().lower(),)).fetchone()
                    if student_record:
                        st.session_state.user = {
                            "name": f"Parent of {student_record[3]}", 
                            "role": "Parent", 
                            "child_id": student_record[1]
                        }
                        st.session_state.logged_in = True
                        st.rerun()
                    else:
                        st.error("Invalid Student ID. Please check your child's ID and try again.")
                conn.close()


# --- NURSE DASHBOARD ---
def nurse_dashboard():
    st.markdown("## 🩺 Nurse Dashboard — Grades 1 to 12 Directory")
    conn = get_connection()

    tab1, tab2, tab3, tab4 = st.tabs([
        "📋 Student Directory", 
        "➕ Add Student Record", 
        "🔍 Search Students",
        "💬 Messages & Composed Inbox"
    ])

    with tab1:
        st.subheader("Select Grade (1 to 12)")
        grades = list(range(1, 13))
        for i in range(0, len(grades), 4):
            row_grades = grades[i:i+4]
            row_cols = st.columns(4)
            for col_idx, g in enumerate(row_grades):
                with row_cols[col_idx]:
                    if st.button(f"Grade {g}", key=f"grade_btn_{g}"):
                        st.session_state.nurse_selected_grade = g
                        st.session_state.nurse_selected_section = None

        if st.session_state.nurse_selected_grade is not None:
            g = st.session_state.nurse_selected_grade
            st.markdown(f"### Grade {g} — Select Section (A, B, C, D, E)")
            sections = ["A", "B", "C", "D", "E"]
            sec_cols = st.columns(5)
            for idx, sec in enumerate(sections):
                count = conn.execute("SELECT COUNT(*) FROM students WHERE grade = ? AND section = ?", (g, sec)).fetchone()[0]
                with sec_cols[idx]:
                    if st.button(f"Sec {sec} ({count})", key=f"sec_btn_{g}_{sec}"):
                        st.session_state.nurse_selected_section = sec

            if st.session_state.nurse_selected_section is not None:
                sec = st.session_state.nurse_selected_section
                st.markdown(f"#### Students in Grade {g}{sec}")
                students = conn.execute("SELECT * FROM students WHERE grade = ? AND section = ? ORDER BY roll_number ASC", (g, sec)).fetchall()

                if students:
                    for s in students:
                        with st.expander(f"Roll No. {s[2]} — {s[3]} (ID: `{s[1]}`)") :
                            col_img, col_info = st.columns([1, 3])
                            with col_img:
                                if s[11] is not None:
                                    st.image(s[11], width=110)
                                else:
                                    st.markdown("*(No Photo Uploaded)*")
                                st.markdown(f"<div style='text-align: center; font-weight: bold; font-size: 14px; margin-top: 4px;'>{s[3]}</div>", unsafe_allow_html=True)
                            with col_info:
                                st.markdown(f"""
                                    - **Student ID:** `{s[1]}`
                                    - **Roll Number:** {s[2]}
                                    - **Blood Group:** {s[6]}
                                    - **Allergies:** {s[7]}
                                    - **Emergency Contact:** {s[8]}
                                    - **Vaccinations:** {s[9]}
                                    - **Medications:** {s[10]}
                                    """)

                            # Edit Record Sub-Expander
                            with st.expander("✏️ Edit Student Health Record"):
                                with st.form(f"nurse_edit_form_{s[0]}"):
                                    edit_name = st.text_input("Student Name", value=s[3])
                                    edit_roll = st.number_input("Roll Number", value=s[2], min_value=1)
                                    edit_blood = st.text_input("Blood Group", value=s[6])
                                    edit_emergency = st.text_input("Emergency Contact", value=s[8])
                                    edit_allergies = st.text_area("Allergies", value=s[7])
                                    edit_vax = st.text_area("Vaccination History", value=s[9])
                                    edit_meds = st.text_area("Medications", value=s[10])
                                    
                                    uploaded_photo_file = st.file_uploader("Upload / Change Student Photo (PNG/JPG)", type=["png", "jpg", "jpeg"], key=f"nurse_photo_{s[0]}")

                                    if st.form_submit_button("Update Record"):
                                        new_id_num = generate_roll_id(edit_roll, s[4], s[5])
                                        
                                        if uploaded_photo_file is not None:
                                            photo_bytes = uploaded_photo_file.read()
                                            conn.execute("""
                                                UPDATE students 
                                                SET name=?, roll_number=?, id_number=?, blood_group=?, allergies=?, emergency_contact=?, vaccination_history=?, medications=?, photo=?
                                                WHERE id=?
                                            """, (edit_name, edit_roll, new_id_num, edit_blood, edit_allergies, edit_emergency, edit_vax, edit_meds, photo_bytes, s[0]))
                                        else:
                                            conn.execute("""
                                                UPDATE students 
                                                SET name=?, roll_number=?, id_number=?, blood_group=?, allergies=?, emergency_contact=?, vaccination_history=?, medications=?
                                                WHERE id=?
                                            """, (edit_name, edit_roll, new_id_num, edit_blood, edit_allergies, edit_emergency, edit_vax, edit_meds, s[0]))
                                        
                                        conn.commit()
                                        st.success("Student health record and photo updated successfully!")
                                        st.rerun()

                            if st.button("Remove Student (Left School)", key=f"nurse_rem_{s[0]}"):
                                conn.execute("DELETE FROM students WHERE id = ?", (s[0],))
                                conn.commit()
                                st.success(f"Removed {s[3]}")
                                st.rerun()
                else:
                    st.info(f"No students registered in Grade {g}{sec}.")

    with tab2:
        st.subheader("Add Student Health Record")
        with st.form("add_student_form"):
            s_name = st.text_input("Student Name (e.g., Isha V Prasad)")
            col_r, col_g, col_s = st.columns(3)
            with col_r:
                s_roll = st.number_input("Roll Number", min_value=1, step=1, value=1)
            with col_g:
                s_grade = st.selectbox("Grade", list(range(1, 13)))
            with col_s:
                s_section = st.selectbox("Section", ["A", "B", "C", "D", "E"])
            
            s_blood = st.text_input("Blood Group")
            s_emergency = st.text_input("Emergency Contact")
            s_allergies = st.text_area("Allergies")
            s_vax = st.text_area("Vaccination History")
            s_meds = st.text_area("Medications")
            
            uploaded_add_photo = st.file_uploader("Upload Student Photo (PNG/JPG)", type=["png", "jpg", "jpeg"], key="nurse_add_photo")

            if st.form_submit_button("Save Student Record"):
                if s_name:
                    s_id_num = generate_roll_id(s_roll, s_grade, s_section)
                    photo_bytes = uploaded_add_photo.read() if uploaded_add_photo is not None else None
                    try:
                        conn.execute(
                            """
                                INSERT INTO students (id_number, roll_number, name, grade, section, blood_group, allergies, emergency_contact, vaccination_history, medications, photo)
                                VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
                            """,
                            (s_id_num, s_roll, s_name, s_grade, s_section, s_blood, s_allergies, s_emergency, s_vax, s_meds, photo_bytes),
                        )
                        conn.commit()
                        st.success(f"Added {s_name} with ID `{s_id_num}` to Grade {s_grade}{s_section}!")
                        st.rerun()
                    except sqlite3.IntegrityError:
                        st.error(f"Error: Roll number {s_roll} already exists in Grade {s_grade}{s_section}.")
                else:
                    st.error("Student Name is required.")

    with tab3:
        st.subheader("🔍 Search Student Database")
        search_query = st.text_input("Search by Name, Student ID, Blood Group, or Allergies", placeholder="e.g. Aarav, gj_1_5a, B+, Peanuts")
        
        if search_query:
            query_pattern = f"%{search_query.strip()}%"
            search_results = conn.execute(
                """
                SELECT * FROM students 
                WHERE name LIKE ? OR id_number LIKE ? OR blood_group LIKE ? OR allergies LIKE ?
                ORDER BY grade ASC, section ASC, roll_number ASC
                """,
                (query_pattern, query_pattern, query_pattern, query_pattern)
            ).fetchall()

            if search_results:
                st.markdown(f"Found **{len(search_results)}** matching student(s):")
                for s in search_results:
                    with st.expander(f"Grade {s[4]}{s[5]} — Roll No. {s[2]} — {s[3]} (ID: `{s[1]}`)") :
                        col_img, col_info = st.columns([1, 3])
                        with col_img:
                            if s[11] is not None:
                                st.image(s[11], width=110)
                            else:
                                st.markdown("*(No Photo Uploaded)*")
                            st.markdown(f"<div style='text-align: center; font-weight: bold; font-size: 14px; margin-top: 4px;'>{s[3]}</div>", unsafe_allow_html=True)
                        with col_info:
                            st.markdown(f"""
                                - **Student ID:** `{s[1]}`
                                - **Grade / Section:** Grade {s[4]}{s[5]}
                                - **Roll Number:** {s[2]}
                                - **Blood Group:** {s[6]}
                                - **Allergies:** {s[7]}
                                - **Emergency Contact:** {s[8]}
                                - **Vaccinations:** {s[9]}
                                - **Medications:** {s[10]}
                                """)
            else:
                st.info("No matching students found.")
        else:
            st.markdown("*(Enter a search term above to find students)*")

    with tab4:
        st.subheader("✉️ Send New Message")
        sel_g = st.selectbox("Filter Grade for Student", list(range(1, 13)), key="nurse_msg_grade")
        sel_sec = st.selectbox("Filter Section", ["A", "B", "C", "D", "E"], key="nurse_msg_sec")
        
        students_in_class = conn.execute("SELECT name FROM students WHERE grade = ? AND section = ?", (sel_g, sel_sec)).fetchall()
        student_names = [s[0] for s in students_in_class]

        with st.form("nurse_send_msg_form"):
            target_student = st.selectbox("Select Student", student_names if student_names else ["No students available in this class"])
            target_audience = st.selectbox("Send To", ["Parent", "Teacher", "All"])
            msg_text = st.text_area("Message / Note")
            
            if st.form_submit_button("Send Message"):
                if target_student and target_student != "No students available in this class":
                    conn.execute(
                        """
                            INSERT INTO messages (sender, grade, section, target, student_name, message, date)
                            VALUES (?, ?, ?, ?, ?, ?, date('now'))
                        """,
                        ("School Nurse", sel_g, sel_sec, target_audience, target_student, msg_text),
                    )
                    conn.commit()
                    st.success(f"Message sent to {target_audience} successfully!")
                    st.rerun()
                else:
                    st.error("Please select a valid student.")

        st.markdown("---")
        st.subheader("📥 Inbox / Received Messages")
        messages = conn.execute("SELECT sender, grade, section, target, student_name, message, date FROM messages").fetchall()
        if messages:
            for m in messages:
                st.markdown(f"""
                    **From:** {m[0]} (Grade {m[1]}{m[2]}) | **Target:** {m[3]} | **Student:** {m[4]} | **Date:** {m[6]}  
                    **Message:** {m[5]}
                    """)
        else:
            st.info("No messages received.")

    conn.close()


# --- TEACHER DASHBOARD ---
def teacher_dashboard():
    st.markdown("## 📚 Teacher Dashboard — Class Roster & Health Records")
    
    with st.container():
        st.markdown("### ⚙️ Select Your Assigned Class")
        col1, col2 = st.columns(2)
        with col1:
            selected_grade = st.selectbox("Select Grade", list(range(1, 13)), index=st.session_state.teacher_grade - 1, key="t_grade_sel")
        with col2:
            sections_list = ["A", "B", "C", "D", "E"]
            selected_section = st.selectbox("Select Section", sections_list, index=sections_list.index(st.session_state.teacher_section), key="t_sec_sel")
        
        st.session_state.teacher_grade = selected_grade
        st.session_state.teacher_section = selected_section

    grade = st.session_state.teacher_grade
    section = st.session_state.teacher_section
    
    st.markdown(f"---")
    st.markdown(f"### Currently Managing: **Grade {grade}{section}**")

    conn = get_connection()
    tab1, tab2, tab3, tab4 = st.tabs([
        "👥 Class Roster", 
        "➕ Add/Remove Students", 
        "🔍 Search Students",
        "💬 Messages & Composed Inbox"
    ])

    with tab1:
        st.subheader(f"Students in Grade {grade}{section}")
        students = conn.execute("SELECT * FROM students WHERE grade = ? AND section = ? ORDER BY roll_number ASC", (grade, section)).fetchall()

        if students:
            for s in students:
                with st.expander(f"Roll No. {s[2]} — {s[3]} (ID: `{s[1]}` | Blood: {s[6]})"):
                    col_img, col_info = st.columns([1, 3])
                    with col_img:
                        if s[11] is not None:
                            st.image(s[11], width=110)
                        else:
                            st.markdown("*(No Photo Uploaded)*")
                        st.markdown(f"<div style='text-align: center; font-weight: bold; font-size: 14px; margin-top: 4px;'>{s[3]}</div>", unsafe_allow_html=True)
                    with col_info:
                        st.markdown(f"""
                            - **Student ID:** `{s[1]}`
                            - **Roll Number:** {s[2]}
                            - **Blood Group:** {s[6]}
                            - **Allergies:** {s[7]}
                            - **Emergency Contact:** {s[8]}
                            - **Vaccinations:** {s[9]}
                            - **Medications:** {s[10]}
                            """)

                    # Edit Record Sub-Expander
                    with st.expander("✏️ Edit Student Health Record"):
                        with st.form(f"teacher_edit_form_{s[0]}"):
                            edit_name = st.text_input("Student Name", value=s[3])
                            edit_roll = st.number_input("Roll Number", value=s[2], min_value=1)
                            edit_blood = st.text_input("Blood Group", value=s[6])
                            edit_emergency = st.text_input("Emergency Contact", value=s[8])
                            edit_allergies = st.text_area("Allergies", value=s[7])
                            edit_vax = st.text_area("Vaccination History", value=s[9])
                            edit_meds = st.text_area("Medications", value=s[10])
                            
                            uploaded_photo_file = st.file_uploader("Upload / Change Student Photo (PNG/JPG)", type=["png", "jpg", "jpeg"], key=f"teacher_photo_{s[0]}")

                            if st.form_submit_button("Update Record"):
                                new_id_num = generate_roll_id(edit_roll, grade, section)
                                
                                if uploaded_photo_file is not None:
                                    photo_bytes = uploaded_photo_file.read()
                                    conn.execute("""
                                        UPDATE students 
                                        SET name=?, roll_number=?, id_number=?, blood_group=?, allergies=?, emergency_contact=?, vaccination_history=?, medications=?, photo=?
                                        WHERE id=?
                                    """, (edit_name, edit_roll, new_id_num, edit_blood, edit_allergies, edit_emergency, edit_vax, edit_meds, photo_bytes, s[0]))
                                else:
                                    conn.execute("""
                                        UPDATE students 
                                        SET name=?, roll_number=?, id_number=?, blood_group=?, allergies=?, emergency_contact=?, vaccination_history=?, medications=?
                                        WHERE id=?
                                    """, (edit_name, edit_roll, new_id_num, edit_blood, edit_allergies, edit_emergency, edit_vax, edit_meds, s[0]))
                                
                                conn.commit()
                                st.success("Student health record and photo updated successfully!")
                                st.rerun()
        else:
            st.info(f"No students found in Grade {grade}{section}. Use the 'Add/Remove Students' tab to add your class list.")

    with tab2:
        st.subheader(f"Manage Students in Grade {grade}{section}")
        with st.form("teacher_add_student"):
            t_name = st.text_input("Student Name")
            t_roll = st.number_input("Roll Number", min_value=1, step=1, value=1)
            t_blood = st.text_input("Blood Group")
            t_emergency = st.text_input("Emergency Contact")
            t_allergies = st.text_area("Allergies")
            t_vax = st.text_area("Vaccination History")
            t_meds = st.text_area("Medications")
            
            uploaded_t_photo = st.file_uploader("Upload Student Photo (PNG/JPG)", type=["png", "jpg", "jpeg"], key="teacher_add_photo")

            if st.form_submit_button("Add Student to My Class"):
                if t_name:
                    t_id_num = generate_roll_id(t_roll, grade, section)
                    photo_bytes = uploaded_t_photo.read() if uploaded_t_photo is not None else None
                    try:
                        conn.execute(
                            """
                                INSERT INTO students (id_number, roll_number, name, grade, section, blood_group, allergies, emergency_contact, vaccination_history, medications, photo)
                                VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
                            """,
                            (t_id_num, t_roll, t_name, grade, section, t_blood, t_allergies, t_emergency, t_vax, t_meds, photo_bytes),
                        )
                        conn.commit()
                        st.success(f"Added {t_name} with ID `{t_id_num}` to Grade {grade}{section}!")
                        st.rerun()
                    except sqlite3.IntegrityError:
                        st.error(f"Error: Roll number {t_roll} already exists in Grade {grade}{section}.")
                else:
                    st.error("Student Name is required.")

        st.markdown("#### Remove Student from Roster")
        students = conn.execute("SELECT id, name FROM students WHERE grade = ? AND section = ?", (grade, section)).fetchall()
        if students:
            s_dict = {s[1]: s[0] for s in students}
            to_remove = st.selectbox("Select Student to Remove", list(s_dict.keys()))
            if st.button("Remove Selected Student"):
                conn.execute("DELETE FROM students WHERE id = ?", (s_dict[to_remove],))
                conn.commit()
                st.success(f"Removed {to_remove}")
                st.rerun()

    with tab3:
        st.subheader("🔍 Search Students")
        search_query = st.text_input("Search by Name, Student ID, Blood Group, or Allergies", placeholder="e.g. Aarav, gj_1_5a, B+", key="teacher_search")
        
        if search_query:
            query_pattern = f"%{search_query.strip()}%"
            search_results = conn.execute(
                """
                SELECT * FROM students 
                WHERE (name LIKE ? OR id_number LIKE ? OR blood_group LIKE ? OR allergies LIKE ?)
                ORDER BY grade ASC, section ASC, roll_number ASC
                """,
                (query_pattern, query_pattern, query_pattern, query_pattern)
            ).fetchall()

            if search_results:
                st.markdown(f"Found **{len(search_results)}** matching student(s):")
                for s in search_results:
                    with st.expander(f"Grade {s[4]}{s[5]} — Roll No. {s[2]} — {s[3]} (ID: `{s[1]}`)") :
                        col_img, col_info = st.columns([1, 3])
                        with col_img:
                            if s[11] is not None:
                                st.image(s[11], width=110)
                            else:
                                st.markdown("*(No Photo Uploaded)*")
                            st.markdown(f"<div style='text-align: center; font-weight: bold; font-size: 14px; margin-top: 4px;'>{s[3]}</div>", unsafe_allow_html=True)
                        with col_info:
                            st.markdown(f"""
                                - **Student ID:** `{s[1]}`
                                - **Grade / Section:** Grade {s[4]}{s[5]}
                                - **Roll Number:** {s[2]}
                                - **Blood Group:** {s[6]}
                                - **Allergies:** {s[7]}
                                - **Emergency Contact:** {s[8]}
                                - **Vaccinations:** {s[9]}
                                - **Medications:** {s[10]}
                                """)
            else:
                st.info("No matching students found.")
        else:
            st.markdown("*(Enter a search term above to search students)*")

    with tab4:
        st.subheader("✉️ Send Message to Nurse / Parent / All")
        students = conn.execute("SELECT name FROM students WHERE grade = ? AND section = ?", (grade, section)).fetchall()
        student_names = [s[0] for s in students]

        with st.form("teacher_msg_form"):
            target_student = st.selectbox("Select Student", student_names if student_names else ["No students available"])
            target_audience = st.selectbox("Send To", ["Nurse", "Parent", "All"])
            msg_text = st.text_area("Message / Note")
            
            if st.form_submit_button("Send Message"):
                if target_student and target_student != "No students available":
                    conn.execute(
                        """
                            INSERT INTO messages (sender, grade, section, target, student_name, message, date)
                            VALUES (?, ?, ?, ?, ?, ?, date('now'))
                        """,
                        (f"Teacher (Grade {grade}{section})", grade, section, target_audience, target_student, msg_text),
                    )
                    conn.commit()
                    st.success(f"Message sent to {target_audience} successfully!")
                    st.rerun()
                else:
                    st.error("Please select a valid student.")

        st.markdown("---")
        st.subheader(f"📥 Inbox / Received Messages for Grade {grade}{section}")
        teacher_messages = conn.execute(
            """
            SELECT sender, target, student_name, message, date 
            FROM messages 
            WHERE grade = ? AND section = ? AND (target = 'Teacher' OR target = 'All')
            """,
            (grade, section)
        ).fetchall()

        if teacher_messages:
            for tm in teacher_messages:
                st.markdown(f"""
                    **From:** {tm[0]} | **Target Group:** {tm[1]} | **Student:** {tm[2]} | **Date:** {tm[4]}  
                    **Message:** {tm[3]}
                    """)
        else:
            st.info("No messages received for your class.")

    conn.close()


# --- PARENT DASHBOARD ---
def parent_dashboard():
    st.markdown("## 👨‍👩‍👧 Parent Health Passport Portal")
    conn = get_connection()

    child_id = st.session_state.user["child_id"]
    student = conn.execute("SELECT * FROM students WHERE id_number = ?", (child_id,)).fetchone()

    if student:
        st.markdown(f"### 📋 Health Passport for: **{student[3]}** (ID: `{student[1]}` | Roll No: {student[2]})")
        
        col_img, col_info = st.columns([1, 3])
        with col_img:
            if student[11] is not None:
                st.image(student[11], width=120)
            else:
                st.markdown("*(No Photo Uploaded)*")
            st.markdown(f"<div style='text-align: center; font-weight: bold; font-size: 15px; margin-top: 4px;'>{student[3]}</div>", unsafe_allow_html=True)
        with col_info:
            st.markdown(f"""
                - **Grade / Section:** Grade {student[4]}{student[5]}
                - **Blood Group:** {student[6]}
                - **Allergies:** {student[7]}
                - **Emergency Contact:** {student[8]}
                - **Vaccination History:** {student[9]}
                - **Medications:** {student[10]}
                """)

        st.markdown("### 📨 Messages & Updates for This Child")
        parent_messages = conn.execute(
            """
            SELECT sender, target, message, date 
            FROM messages 
            WHERE student_name = ? AND (target = 'Parent' OR target = 'All')
            """,
            (student[3],)
        ).fetchall()

        if parent_messages:
            for pm in parent_messages:
                st.markdown(f"""
                    **From:** {pm[0]} | **Audience:** {pm[1]} | **Date:** {pm[3]}  
                    **Message:** {pm[2]}
                    """)
        else:
            st.info("No messages for this student yet.")
    else:
        st.error("Student record not found.")

    conn.close()


# --- MAIN ROUTER ---
def main():
    if not st.session_state.logged_in:
        login_page()
    else:
        st.sidebar.markdown(f"**User:** {st.session_state.user['name']}")
        st.sidebar.markdown(f"**Role:** {st.session_state.user['role']}")
        
        role = st.session_state.user["role"]

        # --- EMERGENCY SOS / QUICK TRIAGE WIDGET IN SIDEBAR ---
        if role in ["Nurse", "Teacher"]:
            st.sidebar.markdown("---")
            with st.sidebar.expander("🚨 EMERGENCY SOS / TRIAGE"):
                st.markdown("Instantly look up life-saving details during an emergency.")
                sos_query = st.text_input("Enter Student Name or ID", placeholder="e.g. Aarav", key="sos_search_input")
                if sos_query:
                    conn_sos = get_connection()
                    sos_pat = f"%{sos_query.strip()}%"
                    sos_res = conn_sos.execute(
                        "SELECT name, grade, section, blood_group, allergies, emergency_contact, medications FROM students WHERE name LIKE ? OR id_number LIKE ?", 
                        (sos_pat, sos_pat)
                    ).fetchall()
                    conn_sos.close()

                    if sos_res:
                        for sr in sos_res:
                            st.markdown(f"""
                            <div class='emergency-box'>
                                <b>⚠️ {sr[0]} (Gr {sr[1]}{sr[2]})</b><br>
                                🩸 <b>Blood Group:</b> {sr[3]}<br>
                                🥜 <b>Allergies:</b> {sr[4]}<br>
                                💊 <b>Medications:</b> {sr[6]}<br>
                                📞 <b>Emergency:</b> <a href="tel:{sr[5]}">{sr[5]}</a>
                            </div>
                            """, unsafe_allow_html=True)
                    else:
                        st.warning("No student found.")

        if st.sidebar.button("Logout"):
            st.session_state.logged_in = False
            st.session_state.user = None
            st.session_state.nurse_selected_grade = None
            st.session_state.nurse_selected_section = None
            st.rerun()

        st.sidebar.markdown("---")
        
        if role == "Nurse":
            nurse_dashboard()
        elif role == "Teacher":
            teacher_dashboard()
        elif role == "Parent":
            parent_dashboard()


if __name__ == "__main__":
    main()
