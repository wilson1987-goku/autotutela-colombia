import streamlit as st
from datetime import datetime, timedelta
import pandas as pd
from docx import Document
from docx.shared import Inches
import os

# ===================== CONFIG =====================
st.set_page_config(page_title="AutoTutela Colombia", layout="wide")
st.title("🛡️ AutoTutela - Automatizador de Contestaciones de Tutela")
st.markdown("**Herramienta auxiliar para entidades y abogados en Colombia** - Revisión humana obligatoria")

# ===================== SIDEBAR =====================
st.sidebar.header("Navegación")
pagina = st.sidebar.selectbox("Ir a", ["Generador de Contestación", "Seguimiento de Tutelas", "Plantillas por Materia", "IA Assistant"])

# Datos comunes
ciudades = ["Bogotá D.C.", "Medellín", "Cali", "Barranquilla", "Bucaramanga", "Otra"]

# ===================== PÁGINA 1: GENERADOR =====================
if pagina == "Generador de Contestación":
    st.header("Generador de Contestación de Tutela")
    
    col1, col2 = st.columns(2)
    with col1:
        ciudad = st.selectbox("Ciudad", ciudades)
        despacho = st.text_input("Despacho Judicial (ej: Juzgado Primero Civil Municipal)")
        radicado = st.text_input("Radicado de la Tutela")
        accionante = st.text_input("Nombre del Accionante")
    
    with col2:
        entidad = st.text_input("Entidad Accionada", value="Mi Entidad")
        responsable = st.text_input("Nombre del Responsable")
        cedula = st.text_input("Cédula")
        cargo = st.text_input("Cargo", value="Jefe Oficina Jurídica")
    
    materia = st.selectbox("Materia principal", ["Salud", "Pensión", "Educación", "Derecho de Petición", "Debido Proceso", "Otra"])
    
    hechos = st.text_area("Hechos (resumen objetivo + correcciones)", height=150)
    argumentos = st.text_area("Argumentos de defensa / Improcedencia", height=150,
                              placeholder="Ej: No existe perjuicio irremediable, subsidiariedad, etc.")
    
    normativa = st.text_area("Normativa específica y jurisprudencia", 
                             value="Art. 86 CP, Decreto 2591/1991, Sentencia T-XXX/AAAA")
    
    if st.button("🚀 Generar Contestación", type="primary"):
        fecha = datetime.now().strftime("%d de %B de %Y")
        
        doc = f"""
{ciudad}, {fecha}

Señores
{despacho}
{ciudad}

E.S.D.

Referencia: Acción de Tutela No. {radicado}
Accionante: {accionante}
Accionada: {entidad}

Asunto: CONTESTACIÓN ACCIÓN DE TUTELA

{responsable}, identificado con cédula de ciudadanía No. {cedula}, en calidad de {cargo}, ante usted con respeto me permito contestar la acción de tutela dentro de los términos legales.

**I. HECHOS**
{hechos}

**II. FUNDAMENTOS DE DERECHO**

1. Procedencia / Improcedencia:
La acción de tutela es improcedente por [subsidiariedad - falta de perjuicio irremediable - etc.] según art. 6 Decreto 2591 de 1991.

2. Análisis de fondo:
{normativa}

{argumentos}

**III. PRUEBAS**
Se adjuntan los siguientes documentos: [lista]

**IV. PETICIÓN**
Solicito al Despacho declarar **IMPROCEDENTE** la acción de tutela, o en subsidio, **NEGAR** las pretensiones del accionante.

Atentamente,

{responsable}
{cargo}
{entidad}
Teléfono / Email: [completar]
        """
        
        st.success("¡Contestación generada!")
        st.text_area("Documento generado", doc, height=500)
        
        # Descargar Word
        if st.button("📥 Descargar como Word"):
            document = Document()
            document.add_paragraph(doc)
            doc_name = f"Contestacion_Tutela_{radicado}.docx"
            document.save(doc_name)
            with open(doc_name, "rb") as f:
                st.download_button("Descargar archivo", f, file_name=doc_name)

# ===================== PÁGINA 2: SEGUIMIENTO =====================
elif pagina == "Seguimiento de Tutelas":
    st.header("Seguimiento de Tutelas")
    
    if 'tutelas' not in st.session_state:
        st.session_state.tutelas = pd.DataFrame(columns=["Radicado", "Accionante", "Materia", "Fecha_Recepcion", "Fecha_Limite", "Estado", "Responsable"])
    
    with st.expander("➕ Nueva Tutela"):
        nuevo = {
            "Radicado": st.text_input("Radicado"),
            "Accionante": st.text_input("Accionante"),
            "Materia": st.selectbox("Materia", ["Salud", "Pensión", ...]),
            "Fecha_Recepcion": st.date_input("Fecha de recepción"),
            "Responsable": st.text_input("Responsable"),
            "Estado": st.selectbox("Estado", ["Pendiente", "En redacción", "Contestada", "Fallada"])
        }
        if st.button("Guardar Tutela"):
            fecha_limite = nuevo["Fecha_Recepcion"] + timedelta(days=10)
            nuevo["Fecha_Limite"] = fecha_limite
            st.session_state.tutelas = pd.concat([st.session_state.tutelas, pd.DataFrame([nuevo])], ignore_index=True)
            st.success("Tutela registrada")
    
    st.dataframe(st.session_state.tutelas, use_container_width=True)
    
    # Alertas de plazos
    if not st.session_state.tutelas.empty:
        vencidas = st.session_state.tutelas[st.session_state.tutelas["Fecha_Limite"] <= datetime.now().date()]
        if not vencidas.empty:
            st.error(f"⚠️ Hay {len(vencidas)} tutelas vencidas o por vencer!")

# ===================== Otras páginas (Plantillas e IA) =====================
elif pagina == "Plantillas por Materia":
    st.header("Plantillas recomendadas por materia")
    materia_sel = st.selectbox("Selecciona materia", ["Derecho a la Salud", "Pensión", "Educación", "Derecho de Petición"])
    
    if materia_sel == "Derecho a la Salud":
        st.markdown("""**Argumentos comunes:**
- Subsidiariedad (EPS debe agotar mecanismos internos)
- No hay perjuicio irremediable si hay tutela previa
- Jurisprudencia: Sentencias T-760/08, T-xxx/202x""")
    
    # Agrega más plantillas según necesites

elif pagina == "IA Assistant":
    st.header("Asistente IA para Tutelas")
    prompt_base = st.text_area("Pega el texto de la tutela o describe el caso")
    
    if st.button("Generar argumentos con IA"):
        st.info("En producción conecta con Groq/OpenAI. Ejemplo de prompt recomendado:")
        st.code("""Eres un abogado experto en derecho constitucional colombiano.
Analiza la siguiente tutela y genera:
1. Resumen de hechos
2. Causales de improcedencia
3. Argumentos de defensa de fondo
4. Jurisprudencia relevante (Corte Constitucional)
Tutela: [pega aquí]""")

st.sidebar.info("⚠️ Recordatorio: Esta herramienta es auxiliar. La responsabilidad final es del abogado (Sentencia T-323/2024 Corte Constitucional).")

# Para correr: streamlit run app_tutelas.py
