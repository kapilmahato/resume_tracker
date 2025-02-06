# resume_tracker
import streamlit as st
import base64
import os
import io
from dotenv import load_dotenv
from PIL import Image
import pdf2image
import google.generativeai as genai

# Load environment variables
load_dotenv()

# Configure Gemini AI
genai.configure(api_key=os.getenv("GEMINI_API_KEY"))

# Fake user credentials (Replace with database authentication)
USER_CREDENTIALS = {
    "admin": "password123",
    "user": "resume2024"
}

# Initialize session state for authentication
if "authenticated" not in st.session_state:
    st.session_state.authenticated = False

# Login function
def login(username, password):
    if username in USER_CREDENTIALS and USER_CREDENTIALS[username] == password:
        st.session_state.authenticated = True
        st.success("✅ Login successful! Redirecting...")
        st.experimental_rerun()
    else:
        st.error("❌ Invalid username or password")

# Logout function
def logout():
    st.session_state.authenticated = False
    st.experimental_rerun()

# Function to process PDF
def input_pdf_setup(uploaded_file):
    if uploaded_file is not None:
        images = pdf2image.convert_from_bytes(uploaded_file.read())
        first_page = images[0]

        # Convert image to bytes
        img_byte_arr = io.BytesIO()
        first_page.save(img_byte_arr, format='JPEG')
        img_byte_arr = img_byte_arr.getvalue()

        pdf_parts = [
            {
                "mime_type": "image/jpeg",
                "data": base64.b64encode(img_byte_arr).decode()
            }
        ]
        return pdf_parts
    else:
        raise FileNotFoundError("No file uploaded")

# Function to generate Gemini AI response
def get_gemini_response(input_text, pdf_content, prompt):
    model = genai.GenerativeModel('gemini-pro-vision')
    response = model.generate_content([input_text, pdf_content, prompt])
    return response.text

# Set Streamlit page configuration
st.set_page_config(page_title="ATS Resume Expert", page_icon="📄")

# If user is not authenticated, show login page
if not st.session_state.authenticated:
    st.title("🔐 Login Page")
    username = st.text_input("Username", key="username")
    password = st.text_input("Password", type="password", key="password")
    
    if st.button("Login"):
        login(username, password)

# If user is authenticated, show ATS system
else:
    st.header("📑 ATS Tracking System")

    input_text = st.text_area("Job Description:", key="input")
    uploaded_file = st.file_uploader("Upload your resume (PDF)...", type=["pdf"])

    if uploaded_file is not None:
        st.write("✅ PDF Uploaded Successfully")

    # Buttons for processing resume
    submit1 = st.button("Tell Me About the Resume")
    submit3 = st.button("Percentage Match")

    # Input Prompts
    input_prompt1 = """
    You are an experienced Technical Human Resource Manager. Your task is to review the provided resume against the job description.
    Please share your professional evaluation on whether the candidate's profile aligns with the role. 
    Highlight the strengths and weaknesses of the applicant in relation to the specified job requirements.
    """

    input_prompt3 = """
    You are a skilled ATS (Applicant Tracking System) scanner with a deep understanding of data science and ATS functionality. 
    Your task is to evaluate the resume against the provided job description. 
    Give me the percentage of match if the resume matches the job description. 
    First, the output should come as percentage, then keywords missing, and finally final thoughts.
    """

    # Process the resume with Gemini AI
    if submit1:
        if uploaded_file is not None:
            pdf_content = input_pdf_setup(uploaded_file)
            response = get_gemini_response(input_text, pdf_content, input_prompt1)
            st.subheader("The Response is:")
            st.write(response)
        else:
            st.write("⚠️ Please upload the resume.")

    elif submit3:
        if uploaded_file is not None:
            pdf_content = input_pdf_setup(uploaded_file)
            response = get_gemini_response(input_text, pdf_content, input_prompt3)
            st.subheader("The Response is:")
            st.write(response)
        else:
            st.write("⚠️ Please upload the resume.")

    # Logout Button
    if st.button("Logout"):
        logout()
