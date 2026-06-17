import streamlit as st
import google.generativeai as genai
from google.generativeai.types import GenerationConfig
import sqlite3
import datetime
import json
import os
import re

st.set_page_config(page_title="Kando Makro Takip", layout="centered")

api_key = os.environ.get("GEMINI_API_KEY")
if api_key:
    genai.configure(api_key=api_key)
else:
    st.error("Lütfen Settings kısmından GEMINI_API_KEY tanımlamasını yap kanka!")

conn = sqlite3.connect("makro_takip.db", check_same_thread=False)
c = conn.cursor()
c.execute('''CREATE TABLE IF NOT EXISTS ogunler_v3 
             (id INTEGER PRIMARY KEY AUTOINCREMENT, tarih TEXT, ogun_turu TEXT, detay_metni TEXT, kalori REAL, protein REAL, karb REAL, yag REAL)''')
conn.commit()

def urunleri_analiz_et(ogun_turu, urun_listesi):
    metin_icerik = ""
    for u in urun_listesi:
        metin_icerik += f"- {u['miktar']} {u['birim']} {u['isim']}\n"
        
    prompt = f"""
    Sen sporcu beslenmesi üzerine uzman bir diyetisyensin. Aşağıdaki öğünün makro değerlerini hesapla.
    Yalnızca şu JSON formatında yanıt ver, başka hiçbir kelime veya açıklama ekleme. Markdown işareti kullanma:
    {{"kalori": 0, "protein": 0, "karb": 0, "yag": 0}}
    
    Malzemeler:
    {metin_icerik}
    """
    try:
        # CRITICAL FIX: Eski v1beta hatasını aşmak için v1 API sürümünü ve güncel flash modelini zorluyoruz
        client = genai.Client(api_key=api_key) if hasattr(genai, 'Client') else None
        
        if client:
            # Eğer kütüphane yeniyse bu modern yöntem çalışacak
            response = client.models.generate_content(
                model='gemini-1.5-flash',
                contents=prompt,
            )
            text_data = response.text
        else:
            # Eğer kütüphane eskiyse v1 sürümünü el yardımıyla zorlayan tek yöntem bud asıl sihirdir
            model = genai.GenerativeModel(
                model_name="models/gemini-1.5-flash",
                generation_config={"response_mime_type": "application/json"}
            )
            # v1beta yerine doğrudan kararlı v1 api yolunu tetikliyoruz
            import google.ai.generativelanguage as gag
            genai.client._client_manager.default_options = {"api_version": "v1"}
            response = model.generate_content(prompt)
            text_data = response.text
        
        # JSON temizleme filtresi
        text_data = text_data.strip()
        match = re.search(r'\{.*?\}', text_data, re.DOTALL)
        
        if match:
            veri = json.loads(match.group(0))
            return {
                "kalori": float(veri.get("kalori", 0)),
                "protein": float(veri.get("protein", 0)),
                "karb": float(veri.get("karb", 0)),
                "yag": float(veri.get("yag", 0))
            }, metin_icerik, None
        else:
            return {"kalori": 0, "protein": 0, "karb": 0, "yag": 0}, metin_icerik, "Yapay zeka JSON formatında yanıt vermedi."
            
    except Exception as e:
        return {"kalori": 0, "protein": 0, "karb": 0, "yag": 0}, metin_icerik, str(e)

st.title("🏃‍♂️ Kando Kontrollü Makro Takip")

col_tarih, col_tur = st.columns(2)
with col_tarih:
    secilen_tarih = st.date_input("📅 Gün Seç:", datetime.date.today()).strftime("%Y-%m-%d")
with col_tur:
    ogun_turu = st.selectbox("🍽️ Öğün Seç:", ["Kahvaltı", "Öğle", "Akşam", "Atıştırmalık"])

st.markdown("---")
st.subheader("📝 Öğün İçeriğini Gir")

if "urun_sayisi" not in st.session_state:
    st.session_state.urun_sayisi = 1

aktif_urunler = []

for i in range(st.session_state.urun_sayisi):
    st.markdown(f"**Ürün #{i+1}**")
    c1, c2, c3 = st.columns([2, 1, 1])
    
    with c1:
        u_isim = st.text_input(f"Ne yedin / içtin?", key=f"isim_{i}")
    with c2:
        u_birim = st.selectbox("Birimi:", ["gram", "Adet", "Dilim", "Porsiyon", "ml"], key=f"birim_{i}")
    with c3:
        u_miktar = st.number_input("Miktar:", min_value=0.0, step=0.5, format="%.1f", key=f"miktar_{i}")
        
    if u_isim:
        aktif_urunler.append({"isim": u_isim, "birim": u_birim, "miktar": u_miktar})

col_btn1, col_btn2 = st.columns(2)
with col_btn1:
    if st.button("➕ Yeni Ürün Maddesi Ekle"):
        st.session_state.urun_sayisi += 1
        st.rerun()
with col_btn2:
    if st.button("🗑️ Son Maddeyi Kaldır") and st.session_state.urun_sayisi > 1:
        st.session_state.urun_sayisi -= 1
        st.rerun()

st.markdown("---")

if st.button("🚀 Bu Öğünü Deftere İşle"):
    if aktif_urunler:
        with st.spinner("Yapay zeka hesaplıyor..."):
            makrolar, detay_metni, hata_mesaji = urunleri_analiz_et(ogun_turu, aktif_urunler)
            
            if hata_mesaji:
                st.error(f"🤖 Hata Detayı: {hata_mesaji}")
            else:
                c.execute("""INSERT INTO ogunler_v3 (tarih, ogun_turu, detay_metni, kalori, protein, karb, yag) 
                             VALUES (?, ?, ?, ?, ?, ?, ?)""",
                          (secilen_tarih, ogun_turu, detay_metni, 
                           float(makrolar['kalori']), float(makrolar['protein']), float(makrolar['karb']), float(makrolar['yag'])))
                conn.commit()
                st.session_state.urun_sayisi = 1
                st.success("Öğün başarıyla kaydedildi kando!")
                st.rerun()
    else:
        st.warning("Lütfen önce en az bir ürün ismi gir kanka.")

st.markdown("---")
st.subheader(f"📊 {secilen_tarih} Tarihli Raporun")

c.execute("SELECT ogun_turu, detay_metni, kalori, protein, karb, yag FROM ogunler_v3 WHERE tarih = ?", (secilen_tarih,))
gunun_verileri = c.fetchall()

if gunun_verileri:
    toplam_kalori = sum(x[2] for x in gunun_verileri)
    toplam_protein = sum(x[3] for x in gunun_verileri)
    toplam_karb = sum(x[4] for x in gunun_verileri)
    toplam_yag = sum(x[5] for x in gunun_verileri)
    
    v1, v2, v3, v4 = st.columns(4)
    v1.metric("🔥 Toplam Kalori", f"{int(toplam_kalori)} kcal")
    v2.metric("🥩 Protein", f"{int(toplam_protein)} g")
    v3.metric("🍞 Karbonhidrat", f"{int(toplam_karb)} g")
    v4.metric("🥑 Yağ", f"{int(toplam_yag)} g")
    
    st.markdown("### 📝 Bugün Kaydedilen Öğünler")
    for ogun in gunun_verileri:
        with st.expander(f"🟢 {ogun[0]} - {int(ogun[2])} kcal"):
            st.write(f"**İçerik:**\n{ogun[1]}")
            st.write(f"**Makro Detayı:** {int(ogun[3])}g Protein | {int(ogun[4])}g Karb | {int(ogun[5])}g Yağ")
else:
    st.info("Bu tarihte henüz bir kayıt yok kando.")
