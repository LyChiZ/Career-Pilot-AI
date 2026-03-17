"""
CV Builder — NotebookLM-style Streamlit App
============================================
Layout:
  Left   → Upload documents (PDF / DOCX)
  Middle → Polished CV output
  Right  → AI suggestions & reasoning

Dependencies (requirements.txt):
  streamlit>=1.32.0
  anthropic>=0.25.0
  pypdf>=4.0.0
  python-docx>=1.1.0
"""

import io
import streamlit as st
import anthropic
import pypdf
import docx

# ── Page config ─────────────────────────────────────────────────────────────
st.set_page_config(
    page_title="CV Builder",
    page_icon="📄",
    layout="wide",
    initial_sidebar_state="collapsed",
)

# ── Custom CSS — dark theme inspired by NotebookLM ───────────────────────────
st.markdown("""
<style>
/* Import fonts */
@import url('https://fonts.googleapis.com/css2?family=DM+Serif+Display:ital@0;1&family=DM+Sans:wght@300;400;500;600&display=swap');

/* Global reset */
html, body, [class*="css"] {
    font-family: 'DM Sans', sans-serif;
    background-color: #0f1117;
    color: #e8e8e8;
}

/* Hide default Streamlit header & footer */
#MainMenu, footer, header { visibility: hidden; }

/* Remove default top padding */
.block-container { padding-top: 1.2rem; padding-bottom: 1rem; }

/* App header bar */
.app-header {
    display: flex;
    align-items: center;
    gap: 12px;
    padding: 10px 0 18px 0;
    border-bottom: 1px solid #2a2d3a;
    margin-bottom: 1rem;
}
.app-header h1 {
    font-family: 'DM Serif Display', serif;
    font-size: 1.6rem;
    font-weight: 400;
    color: #ffffff;
    margin: 0;
}
.app-header .badge {
    background: #3b5bdb;
    color: #fff;
    font-size: 0.65rem;
    font-weight: 600;
    padding: 2px 8px;
    border-radius: 20px;
    letter-spacing: 0.06em;
    text-transform: uppercase;
}

/* Column card containers */
.panel {
    background: #161b27;
    border: 1px solid #252a38;
    border-radius: 12px;
    padding: 1.2rem 1.1rem;
    min-height: 80vh;
}
.panel-title {
    font-family: 'DM Serif Display', serif;
    font-size: 1rem;
    font-weight: 400;
    letter-spacing: 0.02em;
    color: #a0aabf;
    margin-bottom: 1rem;
    text-transform: uppercase;
    font-size: 0.72rem;
    letter-spacing: 0.12em;
}

/* File uploader override */
[data-testid="stFileUploader"] {
    background: #1e2330;
    border: 1.5px dashed #3b4260;
    border-radius: 10px;
    padding: 0.6rem;
}
[data-testid="stFileUploader"]:hover {
    border-color: #4c6ef5;
}

/* Uploaded file chips */
.file-chip {
    display: flex;
    align-items: center;
    gap: 8px;
    background: #1e2330;
    border: 1px solid #2e3450;
    border-radius: 8px;
    padding: 8px 12px;
    margin: 6px 0;
    font-size: 0.82rem;
    color: #c5cde8;
}
.file-chip .icon { font-size: 1rem; }

/* CV output area */
.cv-output {
    background: #1a1f2e;
    border-radius: 10px;
    padding: 1.5rem 1.8rem;
    font-family: 'DM Sans', sans-serif;
    font-size: 0.9rem;
    line-height: 1.75;
    color: #dde3f0;
    white-space: pre-wrap;
    border: 1px solid #252d45;
    min-height: 60vh;
    max-height: 70vh;
    overflow-y: auto;
}

/* Suggestions panel */
.suggestion-card {
    background: #1a1f2e;
    border-left: 3px solid #4c6ef5;
    border-radius: 0 8px 8px 0;
    padding: 10px 14px;
    margin: 10px 0;
    font-size: 0.84rem;
    color: #c5cde8;
    line-height: 1.6;
}
.suggestion-card .label {
    font-size: 0.7rem;
    color: #4c6ef5;
    font-weight: 600;
    text-transform: uppercase;
    letter-spacing: 0.08em;
    margin-bottom: 4px;
}
.suggestion-area {
    max-height: 70vh;
    overflow-y: auto;
}

/* Primary button */
.stButton > button {
    background: #3b5bdb;
    color: white;
    border: none;
    border-radius: 8px;
    font-family: 'DM Sans', sans-serif;
    font-weight: 500;
    font-size: 0.88rem;
    padding: 0.5rem 1.2rem;
    width: 100%;
    transition: background 0.2s;
}
.stButton > button:hover { background: #4c6ef5; border: none; }
.stButton > button:active { background: #2f4ac7; }

/* Dividers */
hr { border-color: #252a38; margin: 1rem 0; }

/* Scrollbar styling */
::-webkit-scrollbar { width: 5px; }
::-webkit-scrollbar-track { background: #161b27; }
::-webkit-scrollbar-thumb { background: #2e3a5a; border-radius: 4px; }

/* Status / info boxes */
.stAlert { border-radius: 8px; }

/* Spinner */
[data-testid="stSpinner"] { color: #4c6ef5; }
</style>
""", unsafe_allow_html=True)

# ── Helpers ──────────────────────────────────────────────────────────────────

def extract_text_from_pdf(file_bytes: bytes) -> str:
    reader = pypdf.PdfReader(io.BytesIO(file_bytes))
    return "\n".join(page.extract_text() or "" for page in reader.pages)


def extract_text_from_docx(file_bytes: bytes) -> str:
    doc = docx.Document(io.BytesIO(file_bytes))
    return "\n".join(para.text for para in doc.paragraphs)


def extract_text(uploaded_file) -> str:
    raw = uploaded_file.read()
    name = uploaded_file.name.lower()
    if name.endswith(".pdf"):
        return extract_text_from_pdf(raw)
    elif name.endswith(".docx") or name.endswith(".doc"):
        return extract_text_from_docx(raw)
    else:
        try:
            return raw.decode("utf-8")
        except Exception:
            return ""


def build_prompt(combined_text: str, target_role: str, tone: str) -> str:
    return f"""You are an expert CV writer and career coach. A user has provided raw documents 
(CV, certificates, awards, project overviews) and wants a polished, ready-to-send CV.

Target role / industry: {target_role or "General / Not specified"}
Preferred tone: {tone}

--- RAW DOCUMENTS ---
{combined_text[:12000]}
--- END OF DOCUMENTS ---

Your task has TWO parts. Reply in this EXACT format:

===CV_START===
[Write a complete, polished, ATS-friendly CV in plain text here. 
Use clear sections: Summary, Work Experience, Education, Skills, Certifications & Awards, Projects.
Use clean formatting with section headers in ALL CAPS followed by a line of dashes.
Do NOT use markdown bold (**) — use plain text only.]
===CV_END===

===SUGGESTIONS_START===
[List every change you made as structured items. For each item use this format:
CHANGE: <short title>
REASON: <one or two sentence explanation>
---
List at minimum 5 changes.]
===SUGGESTIONS_END===
"""


def call_claude(prompt: str, api_key: str) -> tuple[str, str]:
    """Returns (polished_cv, suggestions)."""
    client = anthropic.Anthropic(api_key=api_key)
    message = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=4096,
        messages=[{"role": "user", "content": prompt}],
    )
    full = message.content[0].text

    cv_text = ""
    suggestions_text = ""

    if "===CV_START===" in full and "===CV_END===" in full:
        cv_text = full.split("===CV_START===")[1].split("===CV_END===")[0].strip()

    if "===SUGGESTIONS_START===" in full and "===SUGGESTIONS_END===" in full:
        suggestions_text = (
            full.split("===SUGGESTIONS_START===")[1]
            .split("===SUGGESTIONS_END===")[0]
            .strip()
        )

    return cv_text, suggestions_text


def parse_suggestions(raw: str) -> list[dict]:
    """Parse suggestions block into list of {change, reason} dicts."""
    items = []
    blocks = raw.split("---")
    for block in blocks:
        block = block.strip()
        if not block:
            continue
        change, reason = "", ""
        for line in block.splitlines():
            if line.startswith("CHANGE:"):
                change = line.replace("CHANGE:", "").strip()
            elif line.startswith("REASON:"):
                reason = line.replace("REASON:", "").strip()
        if change:
            items.append({"change": change, "reason": reason})
    return items


# ── Session state ─────────────────────────────────────────────────────────────
for key in ("cv_output", "suggestions_raw", "files_text"):
    if key not in st.session_state:
        st.session_state[key] = ""

# ── Header ────────────────────────────────────────────────────────────────────
st.markdown("""
<div class="app-header">
    <span style="font-size:1.5rem">📄</span>
    <h1>CV Builder</h1>
    <span class="badge">AI-Powered</span>
</div>
""", unsafe_allow_html=True)

# ── Three-column layout ───────────────────────────────────────────────────────
left, mid, right = st.columns([1.1, 1.8, 1.1], gap="medium")

# ════════════════════════════════════════════════════════════════════════════
# LEFT PANEL — Upload & settings
# ════════════════════════════════════════════════════════════════════════════
with left:
    st.markdown('<div class="panel">', unsafe_allow_html=True)
    st.markdown('<div class="panel-title">📁 Documents</div>', unsafe_allow_html=True)

    uploaded_files = st.file_uploader(
        "Upload your documents",
        type=["pdf", "docx", "doc", "txt"],
        accept_multiple_files=True,
        label_visibility="collapsed",
    )

    if uploaded_files:
        st.markdown("**Uploaded files:**")
        for f in uploaded_files:
            icon = "📄" if f.name.endswith(".pdf") else "📝"
            size_kb = round(f.size / 1024, 1)
            st.markdown(
                f'<div class="file-chip"><span class="icon">{icon}</span>'
                f'{f.name} <span style="color:#555;margin-left:auto">{size_kb} KB</span></div>',
                unsafe_allow_html=True,
            )

    st.markdown("<hr>", unsafe_allow_html=True)
    st.markdown('<div class="panel-title">⚙️ Options</div>', unsafe_allow_html=True)

    api_key = st.text_input(
        "Anthropic API Key",
        type="password",
        placeholder="sk-ant-...",
        help="Your key is never stored. Get one at console.anthropic.com",
    )

    target_role = st.text_input(
        "Target Role / Industry",
        placeholder="e.g. Software Engineer, Finance, Marketing",
    )

    tone = st.selectbox(
        "CV Tone",
        ["Professional & Concise", "Academic & Detailed", "Creative & Engaging", "Executive & Strategic"],
    )

    st.markdown("<br>", unsafe_allow_html=True)
    generate_btn = st.button("✨ Generate Polished CV", use_container_width=True)
    st.markdown('</div>', unsafe_allow_html=True)

# ════════════════════════════════════════════════════════════════════════════
# MIDDLE PANEL — CV output
# ════════════════════════════════════════════════════════════════════════════
with mid:
    st.markdown('<div class="panel">', unsafe_allow_html=True)
    st.markdown('<div class="panel-title">📋 Polished CV</div>', unsafe_allow_html=True)

    # ── Handle generation ─────────────────────────────────────────────────
    if generate_btn:
        if not uploaded_files:
            st.error("Please upload at least one document first.")
        elif not api_key:
            st.error("Please enter your Anthropic API key.")
        else:
            with st.spinner("Reading documents and crafting your CV…"):
                combined = ""
                for f in uploaded_files:
                    text = extract_text(f)
                    combined += f"\n\n[SOURCE: {f.name}]\n{text}"

                prompt = build_prompt(combined, target_role, tone)
                try:
                    cv_out, suggestions_out = call_claude(prompt, api_key)
                    st.session_state.cv_output = cv_out
                    st.session_state.suggestions_raw = suggestions_out
                except anthropic.AuthenticationError:
                    st.error("Invalid API key. Please check and try again.")
                except anthropic.APIError as e:
                    st.error(f"API error: {e}")

    # ── Display CV ────────────────────────────────────────────────────────
    if st.session_state.cv_output:
        st.markdown(
            f'<div class="cv-output">{st.session_state.cv_output}</div>',
            unsafe_allow_html=True,
        )
        st.download_button(
            label="⬇️ Download CV as .txt",
            data=st.session_state.cv_output,
            file_name="polished_cv.txt",
            mime="text/plain",
            use_container_width=True,
        )
    else:
        st.markdown(
            '<div class="cv-output" style="display:flex;align-items:center;'
            'justify-content:center;color:#3b4260;font-style:italic;">'
            'Your polished CV will appear here after generation.</div>',
            unsafe_allow_html=True,
        )

    st.markdown('</div>', unsafe_allow_html=True)

# ════════════════════════════════════════════════════════════════════════════
# RIGHT PANEL — Suggestions
# ════════════════════════════════════════════════════════════════════════════
with right:
    st.markdown('<div class="panel">', unsafe_allow_html=True)
    st.markdown('<div class="panel-title">💡 AI Suggestions</div>', unsafe_allow_html=True)

    if st.session_state.suggestions_raw:
        items = parse_suggestions(st.session_state.suggestions_raw)
        if items:
            st.markdown('<div class="suggestion-area">', unsafe_allow_html=True)
            for i, item in enumerate(items, 1):
                st.markdown(
                    f'<div class="suggestion-card">'
                    f'<div class="label">Change {i} · {item["change"]}</div>'
                    f'{item["reason"]}'
                    f'</div>',
                    unsafe_allow_html=True,
                )
            st.markdown('</div>', unsafe_allow_html=True)
        else:
            st.markdown(
                '<div class="cv-output">' + st.session_state.suggestions_raw + '</div>',
                unsafe_allow_html=True,
            )
    else:
        st.markdown(
            '<div class="cv-output" style="display:flex;align-items:center;'
            'justify-content:center;color:#3b4260;font-style:italic;">'
            'AI changes & reasoning will appear here.</div>',
            unsafe_allow_html=True,
        )

    st.markdown('</div>', unsafe_allow_html=True)
