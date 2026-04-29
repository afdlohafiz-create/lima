import streamlit as st
from rdkit import Chem
from rdkit.Chem import AllChem, Descriptors
from stmol import showmol
import py3Dmol

# Konfigurasi Halaman
st.set_page_config(page_title="AI ChemVisualizer Pro", layout="wide")

# --- CSS Custom ---
st.markdown("""
    <style>
    .main { background-color: #f5f7f9; }
    .stMetric { background-color: #ffffff; padding: 15px; border-radius: 10px; box-shadow: 0 2px 4px rgba(0,0,0,0.05); }
    </style>
    """, unsafe_allow_html=True)

# --- SIDEBAR: KONTROL VISUAL ---
st.sidebar.header("🎨 Pengaturan Visual")
style_type = st.sidebar.selectbox("Gaya Molekul", ["stick", "sphere", "line", "cross"])
bg_color = st.sidebar.color_picker("Warna Latar", "#ffffff")
spin_mode = st.sidebar.checkbox("Animasi Berputar (Spin)", value=False)

st.sidebar.divider()
st.sidebar.success("✅ Mode Offline Aktif: Asisten Tutor berjalan tanpa API Key.")

# --- HEADER ---
st.title("🧪 AI ChemVisualizer Pro")
st.markdown("Aplikasi ini mengubah kode *SMILES* menjadi struktur *3D Interaktif* dan dilengkapi *Tutor Cerdas Offline*.")

# --- INPUT AREA ---
with st.container():
    c1, c2 = st.columns([3, 1])
    with c1:
        smiles_input = st.text_input("Masukkan Kode SMILES Molekul:", "CC(=O)OC1=CC=CC=C1C(=O)O")
        st.caption("Contoh: CCO (Etanol), C6H6 (Benzena), Aspirin (SMILES di atas)")
    with c2:
        st.write("##")
        if st.button("Generate & Analisis"):
            st.rerun()

# --- FUNGSI RENDERING ---
def make_3d_plot(smiles, style, bg, spin):
    try:
        mol = Chem.MolFromSmiles(smiles)
        mol = Chem.AddHs(mol)
        AllChem.EmbedMolecule(mol, AllChem.ETKDG())
        mblock = Chem.MolToMolBlock(mol)
        
        view = py3Dmol.view(width=800, height=500)
        view.addModel(mblock, 'mol')
        view.setStyle({style: {'colorscheme': 'Jmol', 'radius': 0.3 if style=='stick' else 1.0}})
        view.setBackgroundColor(bg)
        if spin:
            view.animate({'loop': 'backAndForth', 'step': 1})
        view.zoomTo()
        return view, mol
    except:
        return None, None

# --- MAIN DISPLAY: VISUALISASI ---
mol_data_global = None # Menyimpan data molekul untuk dipakai chatbot

if smiles_input:
    res_view, mol_data = make_3d_plot(smiles_input, style_type, bg_color, spin_mode)
    mol_data_global = mol_data
    
    if res_view:
        col_left, col_right = st.columns([2, 1])
        
        with col_left:
            st.subheader("🌐 Model 3D Interaktif")
            showmol(res_view, height=500, width=800)
            
        with col_right:
            st.subheader("📊 Analisis Properti")
            mw = Descriptors.MolWt(mol_data)
            logp = Descriptors.MolLogP(mol_data)
            h_donors = Descriptors.NumHDonors(mol_data)
            h_acceptors = Descriptors.NumHAcceptors(mol_data)
            formula = Chem.rdMolDescriptors.CalcMolFormula(mol_data)
            
            st.metric("Rumus Kimia", formula)
            st.metric("Berat Molekul", f"{mw:.2f} g/mol")
            st.metric("LogP (Kelarutan)", f"{logp:.2f}")
            
            with st.expander("Detail Ikatan"):
                st.write(f"🔹 H-Bond Donors: {h_donors}")
                st.write(f"🔹 H-Bond Acceptors: {h_acceptors}")
    else:
        st.error("Gagal memproses molekul. Silakan periksa kembali kode SMILES Anda.")

st.divider()

# --- FUNGSI OTAK CHATBOT OFFLINE ---
def offline_chatbot_brain(pertanyaan, mol):
    if not mol:
        return "Tolong masukkan kode molekul yang valid terlebih dahulu di atas ya."
    
    pertanyaan = pertanyaan.lower()
    
    # Menghitung data real-time
    mw = Descriptors.MolWt(mol)
    formula = Chem.rdMolDescriptors.CalcMolFormula(mol)
    logp = Descriptors.MolLogP(mol)
    
    # Logika deteksi kata kunci
    if "rumus" in pertanyaan or "kimia" in pertanyaan:
        return f"Rumus kimia untuk molekul yang sedang kita lihat adalah *{formula}*."
    
    elif "massa" in pertanyaan or "berat" in pertanyaan:
        return f"Berat molekul (massa molar) dari zat ini adalah sekitar *{mw:.2f} g/mol*."
    
    elif "polar" in pertanyaan or "larut" in pertanyaan or "air" in pertanyaan:
        if logp < 0.5:
            return f"Molekul ini memiliki nilai LogP {logp:.2f}. Karena nilainya rendah, molekul ini cenderung *polar* dan mudah larut dalam air (hidrofilik)."
        elif logp < 3:
            return f"Molekul ini memiliki nilai LogP {logp:.2f}. Sifatnya berada di tengah-tengah, bisa sedikit larut dalam air maupun pelarut organik."
        else:
            return f"Molekul ini memiliki nilai LogP {logp:.2f}. Karena nilainya tinggi, molekul ini cenderung *non-polar* dan lebih mudah larut dalam lemak/minyak (hidrofobik) dibandingkan air."
            
    elif "atom" in pertanyaan or "unsur" in pertanyaan:
        atoms = [atom.GetSymbol() for atom in mol.GetAtoms()]
        komposisi = dict((x, atoms.count(x)) for x in set(atoms))
        return f"Molekul ini tersusun dari atom-atom berikut: {komposisi}."
        
    elif "ikatan" in pertanyaan or "hidrogen" in pertanyaan:
        h_donors = Descriptors.NumHDonors(mol)
        h_acceptors = Descriptors.NumHAcceptors(mol)
        return f"Molekul ini memiliki *{h_donors}* donor ikatan hidrogen dan *{h_acceptors}* akseptor ikatan hidrogen."
        
    else:
        return "Maaf, sebagai asisten offline, saya hanya bisa menjawab pertanyaan seputar: *rumus kimia, massa molekul, kepolaran (kelarutan), komposisi atom, dan ikatan hidrogen* dari molekul yang tampil. Ada yang ingin ditanyakan dari topik tersebut?"

# --- BAGIAN CHATBOT UI ---
st.subheader("💬 Asisten Tutor Kimia (Offline)")
st.write("Tanyakan seputar massa, rumus, kepolaran, atau ikatan dari molekul di atas.")

if "chat_history" not in st.session_state:
    st.session_state.chat_history = [
        {"role": "assistant", "content": "Halo! Saya adalah asisten tutor kimia lokal Anda. Coba tanyakan: 'Apakah molekul ini polar?' atau 'Berapa massanya?'"}
    ]

# Menampilkan riwayat chat
for message in st.session_state.chat_history:
    with st.chat_message(message["role"]):
        st.markdown(message["content"])

# Input Chat
if user_question := st.chat_input("Contoh: Berapa massa molekul ini?"):
    
    # Tampilkan teks user
    with st.chat_message("user"):
        st.markdown(user_question)
    st.session_state.chat_history.append({"role": "user", "content": user_question})

    # Proses jawaban cerdas buatan (Rule-based)
    with st.chat_message("assistant"):
        ai_reply = offline_chatbot_brain(user_question, mol_data_global)
        st.markdown(ai_reply)
        st.session_state.chat_history.append({"role": "assistant", "content": ai_reply})

st.divider()
st.markdown("<div style='text-align: center'><p style='font-size: 0.8em'>Proyek TIK Pembelajaran Kimia © 2026</p></div>", unsafe_allow_html=True)