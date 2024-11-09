import tkinter as tk
from tkinter import messagebox, simpledialog
import sqlite3
from datetime import datetime
import matplotlib.pyplot as plt
import shutil

# Classe para o banco de dados
class Banco:
    def __init__(self):
        self.conn, self.cursor = self.conectar_banco()

    def conectar_banco(self):
        try:
            conn = sqlite3.connect('meubanco.db')
            cursor = conn.cursor()
            return conn, cursor
        except sqlite3.Error as e:
            messagebox.showerror("Erro", f"Erro ao conectar ao banco de dados: {e}")
            return None, None

    def criar_tabela(self):
        self.cursor.execute("""
            CREATE TABLE IF NOT EXISTS registros (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                mes INTEGER,
                ano INTEGER,
                semana INTEGER,
                dia INTEGER,
                fechamento REAL,
                servico REAL,
                despesas REAL,
                chaves INTEGER,
                alicates INTEGER,
                total REAL
            )
        """)
        self.conn.commit()

    def inserir_registro(self, registro):
        self.cursor.execute("""
            INSERT INTO registros (mes, ano, semana, dia, fechamento, servico, despesas, chaves, alicates, total)
            VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
        """, (registro.mes, registro.ano, registro.semana, registro.dia, registro.fechamento, registro.servico,
              registro.despesas, registro.chaves, registro.alicates, registro.total))
        self.conn.commit()

    def fechar_conexao(self):
        self.conn.close()

    def buscar_registros_por_dia(self, dia):
        self.cursor.execute("SELECT * FROM registros WHERE dia = ?", (dia,))
        return self.cursor.fetchall()

    def gerar_relatorio(self, mes=None, ano=None, dia=None):
        query = "SELECT SUM(fechamento), SUM(servico), SUM(despesas), SUM(total), SUM(chaves), SUM(alicates) FROM registros"
        params = []

        if mes:
            query += " WHERE mes = ?"
            params.append(mes)
        if ano:
            query += " AND ano = ?" if mes else " WHERE ano = ?"
            params.append(ano)
        if dia:
            query += " AND dia = ?" if mes or ano else " WHERE dia = ?"
            params.append(dia)

        self.cursor.execute(query, tuple(params))
        return self.cursor.fetchone()

    def gerar_backup(self):
        try:
            shutil.copy('meubanco.db', 'backup_meubanco.db')
            messagebox.showinfo("Sucesso", "Backup realizado com sucesso!")
        except Exception as e:
            messagebox.showerror("Erro", f"Erro ao realizar backup: {e}")

# Classe para representar um registro
class Registro:
    def __init__(self, mes, ano, semana, dia, fechamento, servico, despesas, chaves, alicates):
        self.mes = mes
        self.ano = ano
        self.semana = semana
        self.dia = dia
        self.fechamento = fechamento
        self.servico = servico
        self.despesas = despesas
        self.chaves = chaves
        self.alicates = alicates
        self.total = self.calcular_total()

    def calcular_total(self):
        valor_chave = 8
        valor_alicate = 9
        return self.fechamento + self.servico + self.despesas + (self.chaves * valor_chave) + (self.alicates * valor_alicate)

# Funções auxiliares
def validar_numero(prompt, tipo=float, min_val=None, max_val=None):
    while True:
        try:
            valor = tipo(simpledialog.askstring("Input", prompt))
            if (min_val is not None and valor < min_val) or (max_val is not None and valor > max_val):
                messagebox.showerror("Erro", f"Valor deve estar entre {min_val} e {max_val}.")
            else:
                return valor
        except ValueError:
            messagebox.showerror("Erro", "Entrada inválida. Por favor, insira um número válido.")

def validar_data(dia, mes, ano):
    try:
        datetime(ano, mes, dia)
        return True
    except ValueError:
        return False

def obter_dados_usuario():
    mes = validar_numero("Mês (1-12):", tipo=int, min_val=1, max_val=12)
    ano = validar_numero("Ano:", tipo=int)
    semana = validar_numero("Semana (1-7):", tipo=int, min_val=1, max_val=7)
    dia = validar_numero("Dia (1-31):", tipo=int, min_val=1, max_val=31)

    fechamento = validar_numero("Fechamento:", tipo=float)
    servico = validar_numero("Serviço:", tipo=float)
    despesas = validar_numero("Despesas:", tipo=float)
    chaves = validar_numero("Quantidade de Chaves:", tipo=int)
    alicates = validar_numero("Quantidade de Alicates:", tipo=int)

    return mes, ano, semana, dia, fechamento, servico, despesas, chaves, alicates

# Função principal para registrar dados no banco
def registrar_dados():
    mes, ano, semana, dia, fechamento, servico, despesas, chaves, alicates = obter_dados_usuario()
    
    if not validar_data(dia, mes, ano):
        messagebox.showerror("Erro", f"A data {dia}/{mes}/{ano} não é válida.")
        return

    registro = Registro(mes, ano, semana, dia, fechamento, servico, despesas, chaves, alicates)
    banco = Banco()
    banco.inserir_registro(registro)
    banco.fechar_conexao()
    messagebox.showinfo("Sucesso", "Dados registrados com sucesso!")

# Função para gerar relatório
def gerar_relatorio():
    resposta = simpledialog.askstring("Escolha o filtro", "Escolha: 1 - Dia, 2 - Mês, 3 - Ano", parent=root)
    
    if resposta == "1":
        dia = validar_numero("Digite o dia:", tipo=int, min_val=1, max_val=31)
        resultado = banco.gerar_relatorio(dia=dia)
    elif resposta == "2":
        mes = validar_numero("Digite o mês:", tipo=int, min_val=1, max_val=12)
        resultado = banco.gerar_relatorio(mes=mes)
    elif resposta == "3":
        ano = validar_numero("Digite o ano:", tipo=int)
        resultado = banco.gerar_relatorio(ano=ano)
    else:
        messagebox.showerror("Erro", "Opção inválida.")
        return

    if resultado:
        fechamento, servico, despesas, total, chaves, alicates = resultado
        messagebox.showinfo("Relatório", f"Fechamento Total: R$ {fechamento:.2f}\n"
                                         f"Serviço Total: R$ {servico:.2f}\n"
                                         f"Despesas Totais: R$ {despesas:.2f}\n"
                                         f"Total Geral: R$ {total:.2f}\n"
                                         f"Chaves: {chaves}, Alicates: {alicates}")
    else:
        messagebox.showinfo("Relatório", "Nenhum registro encontrado para o filtro selecionado.")

# Função para gerar gráfico
def gerar_grafico():
    banco = Banco()
    dados = banco.cursor.execute("SELECT mes, SUM(total) FROM registros GROUP BY mes").fetchall()
    banco.fechar_conexao()

    if dados:
        meses = [str(d[0]) for d in dados]
        totais = [d[1] for d in dados]

        plt.bar(meses, totais)
        plt.xlabel("Mês")
        plt.ylabel("Total")
        plt.title("Total por Mês")
        plt.show()

# Função para criar a interface gráfica
def criar_interface():
    global root
    root = tk.Tk()
    root.title("Sistema de Registro")
    root.geometry("500x400")

    label = tk.Label(root, text="Bem-vindo ao Sistema de Registro", font=("Arial", 11))
    label.pack(pady=20)

    btn_registrar = tk.Button(root, text="Registrar Dados", width=20, command=registrar_dados)
    btn_registrar.pack(pady=10)

    btn_relatorio = tk.Button(root, text="Gerar Relatório", width=20, command=gerar_relatorio)
    btn_relatorio.pack(pady=10)

    btn_grafico = tk.Button(root, text="Gerar Gráfico", width=20, command=gerar_grafico)
    btn_grafico.pack(pady=10)

    btn_backup = tk.Button(root, text="Backup Banco de Dados", width=20, command=lambda: banco.gerar_backup())
    btn_backup.pack(pady=10)

    # Botão Sair
    btn_sair = tk.Button(root, text="Sair", width=20, command=root.quit)
    btn_sair.pack(pady=10)

    # Inicializa banco e tabela
    global banco
    banco = Banco()
    banco.criar_tabela()

    root.mainloop()

# Iniciar a aplicação
criar_interface()
