import streamlit as st
import pandas as pd
import json

st.set_page_config(page_title="Time Management Assistant", layout="wide")

st.title("🧠 Time Management Assistant (v0)")
st.write("Πρώτη έκδοση: γράφεις schedule και το app το μετατρέπει σε blocks χρόνου.")

# --- Session state για τα time blocks ---
if "blocks" not in st.session_state:
    st.session_state.blocks = []  # list από dicts


# --- ΣΧΟΛΙΟ: Εδώ αργότερα θα μπει κλήση σε LLM ---
def parse_to_blocks(text: str):
    """
    v0: Περιμένει ότι ο χρήστης δίνει JSON list of blocks.
    v1: Θα καλέσουμε LLM που μετατρέπει ελληνικό κείμενο -> JSON.
    
    Schema (για κάθε block):
    {
        "user": "dimitris",
        "date": "2025-12-01",        # YYYY-MM-DD
        "start": "08:00",            # HH:MM
        "end": "13:00",              # HH:MM
        "activity": "Πωλήσεις",
        "category": "Sales",
        "recurrence": null           # ή π.χ. { "type": "monthly", "day_of_month": 20 }
    }
    """
    try:
        data = json.loads(text)
        if isinstance(data, dict):
            data = [data]
        assert isinstance(data, list)
        return data
    except Exception:
        st.error("Σε αυτή την πρώτη έκδοση, δώσε έγκυρο JSON (list of blocks). "
                 "Στο επόμενο βήμα θα το κάνουμε αυτόματα με LLM.")
        return []


# --- Layout: αριστερά input, δεξιά προβολή ---
col_input, col_view = st.columns([2, 3])

with col_input:
    st.subheader("📝 Είσοδος (φυσική γλώσσα → αργότερα LLM)")
    st.markdown(
        """
        **Τελικός στόχος:** να γράφεις κάτι όπως:
        
        > "Τη Δευτέρα 01/12/2025 8 με 1 θα δουλέψω πωλήσεις και από τις 4 μέχρι τις 6 procurement.  
        > Κάθε μήνα στις 20 θα κάνω έλεγχο πιστωτικών."
        
        και το σύστημα να το μετατρέπει σε blocks.
        
        **Στο v0** όμως, για να ελέγξουμε το flow, βάζουμε κατευθείαν JSON.
        Παράδειγμα:
        ```json
        [
          {
            "user": "dimitris",
            "date": "2025-12-01",
            "start": "08:00",
            "end": "13:00",
            "activity": "Πωλήσεις",
            "category": "Sales",
            "recurrence": null
          },
          {
            "user": "dimitris",
            "date": "2025-12-01",
            "start": "16:00",
            "end": "18:00",
            "activity": "Procurement",
            "category": "Procurement",
            "recurrence": null
          }
        ]
        ```
        """
    )

    user_text = st.text_area("Βάλε εδώ (προς το παρόν JSON ή κείμενο για debug):", height=250)

    if st.button("➕ Πρόσθεσε στο πλάνο"):
        if user_text.strip():
            new_blocks = parse_to_blocks(user_text)
            if new_blocks:
                st.session_state.blocks.extend(new_blocks)
                st.success(f"Προστέθηκαν {len(new_blocks)} blocks στο πλάνο.")
        else:
            st.warning("Γράψε κάτι πρώτα 🙂")


with col_view:
    st.subheader("📅 Προβολή πλάνου (v0)")
    if st.session_state.blocks:
        df = pd.DataFrame(st.session_state.blocks)
        # Ταξινόμηση για να έχεις μια καλύτερη εικόνα
        sort_cols = [c for c in ["user", "date", "start"] if c in df.columns]
        if sort_cols:
            df = df.sort_values(sort_cols)

        st.dataframe(df, use_container_width=True)

        # Ένα μικρό φίλτρο ανά user/έτος - απλό v0
        if "user" in df.columns:
            users = ["(Όλοι)"] + sorted(df["user"].dropna().unique().tolist())
            selected_user = st.selectbox("Φιλτράρισμα ανά χρήστη:", users)

            if selected_user != "(Όλοι)":
                df = df[df["user"] == selected_user]

        st.markdown("### Σύνοψη ωρών ανά user & activity (γρήγορο check)")
        if {"start", "end"}.issubset(df.columns):
            # Προσπάθεια να βγάλουμε διαφορά ώρας σε ώρες
            try:
                temp = df.copy()
                temp["start_dt"] = pd.to_datetime(temp["date"] + " " + temp["start"])
                temp["end_dt"] = pd.to_datetime(temp["date"] + " " + temp["end"])
                temp["hours"] = (temp["end_dt"] - temp["start_dt"]).dt.total_seconds() / 3600

                group_cols = [c for c in ["user", "activity"] if c in temp.columns]
                summary = temp.groupby(group_cols)["hours"].sum().reset_index()
                st.dataframe(summary, use_container_width=True)
            except Exception as e:
                st.warning(f"Δεν μπόρεσα να υπολογίσω ώρες: {e}")
        else:
            st.info("Πρόσθεσε πεδία start/end για να βγάλουμε ώρες.")
    else:
        st.info("Δεν υπάρχουν ακόμα blocks στο πλάνο. Πρόσθεσε με το κουμπί στα αριστερά.")