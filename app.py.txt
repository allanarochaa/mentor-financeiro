from flask import Flask, render_template, request, redirect
import csv
from datetime import datetime
import os
import matplotlib.pyplot as plt

def frase_do_dia():
    try:
        with open(os.path.join("dados", "frases.txt"), "r", encoding="utf-8") as f:
            linhas = [linha.strip() for linha in f if linha.strip()]

        dia_do_ano = datetime.now().timetuple().tm_yday
        i = (dia_do_ano - 1) * 2

        frase1 = linhas[i] if i < len(linhas) else ""
        frase2 = linhas[i + 1] if i + 1 < len(linhas) else ""

        return f"{frase1}\n{frase2}"
    except Exception as e:
        return f"Erro ao carregar a frase: {e}"

app = Flask(__name__)
caminho_csv = os.path.join("dados", "movimentos.csv")

@app.route("/")
def index():
    frase = frase_do_dia()
    return render_template("index.html", frase=frase)

@app.route("/registrar", methods=["POST"])
def registrar():
    tipo = request.form["tipo"]
    descricao = request.form["descricao"]
    valor = request.form["valor"]

    with open(caminho_csv, "a", newline="", encoding="utf-8") as f:
        writer = csv.writer(f)
        writer.writerow([datetime.now().strftime("%Y-%m-%d"), tipo, descricao, valor])

    return redirect("/")

@app.route("/graficos")
def graficos():
    import pandas as pd
    import uuid

    if not os.path.exists(caminho_csv):
        return "Nenhum dado disponível."

    df = pd.read_csv(caminho_csv)
    if df.empty or "descricao" not in df.columns:
        return "Arquivo CSV vazio ou mal formatado."

    df.columns = [col.lower() for col in df.columns]
    df["valor"] = pd.to_numeric(df["valor"], errors="coerce").fillna(0)

    receitas = df[df["tipo"].str.lower() == "receita"]
    despesas = df[df["tipo"].str.lower() == "despesa"]

    pasta_static = os.path.join("static")
    if not os.path.exists(pasta_static):
        os.makedirs(pasta_static)

    fig, axs = plt.subplots(2, 2, figsize=(10, 8))
    fig.suptitle("Gráficos Financeiros")

    if not despesas.empty:
        despesas_agrupadas = despesas.groupby("descricao")["valor"].sum()
        axs[0, 0].pie(despesas_agrupadas, labels=despesas_agrupadas.index, autopct="%1.1f%%", startangle=90)
        axs[0, 0].set_title("Despesas por Categoria")
    else:
        axs[0, 0].text(0.5, 0.5, "Sem despesas", ha="center")

    if not receitas.empty:
        receitas_agrupadas = receitas.groupby("descricao")["valor"].sum()
        axs[0, 1].pie(receitas_agrupadas, labels=receitas_agrupadas.index, autopct="%1.1f%%", startangle=90)
        axs[0, 1].set_title("Receitas por Categoria")
    else:
        axs[0, 1].text(0.5, 0.5, "Sem receitas", ha="center")

    soma_receitas = receitas["valor"].sum()
    soma_despesas = despesas["valor"].sum()
    axs[1, 0].bar(["Receitas", "Despesas"], [soma_receitas, soma_despesas], color=["green", "red"])
    axs[1, 0].set_title("Total Receitas vs Despesas")

    axs[1, 1].axis("off")
    plt.tight_layout()

    nome_imagem = f"graficos_{uuid.uuid4().hex}.png"
    caminho_imagem = os.path.join(pasta_static, nome_imagem)
    plt.savefig(caminho_imagem)
    plt.close()

    return render_template("graficos.html", imagem=nome_imagem)

@app.route("/relatorio")
def relatorio():
    import pandas as pd

    if not os.path.exists(caminho_csv):
        return "Nenhum dado disponível."

    df = pd.read_csv(caminho_csv)

    if df.empty or "descricao" not in df.columns:
        return "Nenhuma movimentação encontrada."

    df.columns = [col.lower() for col in df.columns]

    dados = df.to_dict(orient="records")
    return render_template("relatorio.html", dados=dados)

@app.route("/voz_web", methods=["POST"])
def voz_web():
    import speech_recognition as sr
    import re
    from pydub import AudioSegment

    if 'audio' not in request.files:
        return "Nenhum áudio recebido."

    audio_file = request.files['audio']

    caminho_temp = os.path.join("temp_audio")
    if not os.path.exists(caminho_temp):
        os.makedirs(caminho_temp)

    caminho_webm = os.path.join(caminho_temp, "entrada.webm")
    caminho_wav = os.path.join(caminho_temp, "entrada.wav")
    audio_file.save(caminho_webm)

    try:
        AudioSegment.from_file(caminho_webm).export(caminho_wav, format="wav")

        reconhecedor = sr.Recognizer()
        with sr.AudioFile(caminho_wav) as source:
            audio = reconhecedor.record(source)

        frase = reconhecedor.recognize_google(audio, language="pt-BR")
        print("Você disse:", frase)

        if "recebi" in frase or "ganhei" in frase:
            tipo = "Receita"
        elif "gastei" in frase or "paguei" in frase or "investi" in frase:
            tipo = "Despesa"
        else:
            return "Tipo não identificado."

        valor_match = re.search(r"(\d+(?:[\.,]\d{1,2})?)", frase)
        if not valor_match:
            return "Valor não encontrado."
        valor = valor_match.group(1).replace(",", ".")

        partes = frase.split("com ")
        if len(partes) < 2:
            return "Descrição não encontrada."
        descricao = partes[-1].strip().capitalize()

        with open(caminho_csv, "a", newline="", encoding="utf-8") as f:
            writer = csv.writer(f)
            writer.writerow([datetime.now().strftime("%Y-%m-%d"), tipo, descricao, valor])

        return "Movimento registrado com sucesso!"

    except Exception as e:
        return f"Erro ao processar o áudio: {e}"

if __name__ == "__main__":
    from os import environ
    app.run(host='0.0.0.0', port=int(environ.get("PORT", 5000)))
