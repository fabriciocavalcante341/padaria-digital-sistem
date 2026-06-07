# padaria-digital-sistem
padaria leao da tribo digital 
import tkinter as tk
from tkinter import ttk, messagebox
import sqlite3
from datetime import datetime, timedelta

class SistemaEstoquePadaria:
    def __init__(self, root):
        self.root = root
        self.root.title("🍞 Sistema de Estoque - Padaria Trigo de Ouro")
        self.root.geometry("900x600")
        self.root.configure(bg="#F5F5DC") # Fundo bege claro (estilo padaria)
        
        self.conectar_banco()
        self.criar_interface()
        self.atualizar_tabela()

    def conectar_banco(self):
        """Cria o banco de dados local e a tabela de produtos se não existirem."""
        self.conn = sqlite3.connect("estoque_padaria.db")
        self.cursor = self.conn.cursor()
        self.cursor.execute("""
            CREATE TABLE IF NOT EXISTS estoque (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                produto TEXT NOT NULL,
                quantidade REAL NOT NULL,
                unidade TEXT NOT NULL,
                estoque_minimo REAL NOT NULL,
                validade TEXT NOT NULL
            )
        """)
        self.conn.commit()

    def criar_interface(self):
        # Estilos para os alertas visuais
        style = ttk.Style()
        style.theme_use("clam")
        
        # Cores para alertas na tabela
        style.map("Treeview", background=[('selected', '#0078d7')])
        
        # --- PAINEL DE CADASTRO (LADO ESQUERDO) ---
        lbl_frame = tk.LabelFrame(self.root, text=" Cadastrar / Atualizar Insumo ", bg="#F5F5DC", fg="#5C3A21", font=("Arial", 12, "bold"))
        lbl_frame.pack(side=tk.LEFT, fill=tk.Y, padx=15, pady=15)

        # Campos de entrada
        campos = [
            ("Nome do Insumo:", "txt_produto"),
            ("Quantidade Atual:", "txt_qtd"),
            ("Unidade (kg, L, un):", "cb_unidade"),
            ("Estoque Mínimo:", "txt_min"),
            ("Validade (DD/MM/AAAA):", "txt_validade")
        ]

        self.inputs = {}
        for idx, (label_text, var_name) in enumerate(campos):
            lbl = tk.Label(lbl_frame, text=label_text, bg="#F5F5DC", fg="#5C3A21", font=("Arial", 10, "bold"))
            lbl.grid(row=idx*2, column=0, sticky=tk.W, padx=10, pady=(10, 2))
            
            if var_name == "cb_unidade":
                entry = ttk.Combobox(lbl_frame, values=["kg", "L", "un", "g", "caixa"], width=22)
            else:
                entry = tk.Entry(lbl_frame, font=("Arial", 10), width=25)
                
            entry.grid(row=idx*2+1, column=0, padx=10, pady=(0, 10))
            self.inputs[var_name] = entry

        # Botões de Ação
        btn_salvar = tk.Button(lbl_frame, text="💾 Salvar no Sistema", bg="#4CAF50", fg="white", font=("Arial", 11, "bold"), command=self.salvar_produto, width=20)
        btn_salvar.grid(row=10, column=0, pady=(20, 10), padx=10)

        btn_deletar = tk.Button(lbl_frame, text="❌ Remover Selecionado", bg="#F44336", fg="white", font=("Arial", 11, "bold"), command=self.remover_produto, width=20)
        btn_deletar.grid(row=11, column=0, pady=5, padx=10)
        
        # --- LEGENDA DE ALERTAS ---
        legenda_frame = tk.LabelFrame(lbl_frame, text=" Alertas Visuais ", bg="#F5F5DC", fg="#5C3A21", font=("Arial", 9, "bold"))
        legenda_frame.grid(row=12, column=0, pady=(20, 0), padx=10, sticky=tk.W+tk.E)
        
        tk.Label(legenda_frame, text="🔴 Vencido", bg="#FFCDD2", width=18, anchor="w", padx=5).pack(pady=2)
        tk.Label(legenda_frame, text="🟡 Vence em até 7 dias", bg="#FFF9C4", width=18, anchor="w", padx=5).pack(pady=2)
        tk.Label(legenda_frame, text="🟠 Estoque Baixo (Ruptura)", bg="#FFE0B2", width=18, anchor="w", padx=5).pack(pady=2)

        # --- TABELA DE VISUALIZAÇÃO (LADO DIREITO) ---
        tab_frame = tk.Frame(self.root, bg="#F5F5DC")
        tab_frame.pack(side=tk.RIGHT, fill=tk.BOTH, expand=True, padx=15, pady=15)

        lbl_titulo = tk.Statusbar = tk.Label(tab_frame, text="📋 PAINEL DE MONITORAMENTO DE ESTOQUE", bg="#5C3A21", fg="white", font=("Arial", 12, "bold"), pady=8)
        lbl_titulo.pack(fill=tk.X)

        # Configuração da Treeview (Tabela)
        colunas = ("id", "produto", "quantidade", "unidade", "minimo", "validade")
        self.tabela = ttk.Treeview(tab_frame, columns=colunas, show="headings")
        
        self.tabela.heading("id", text="ID")
        self.tabela.heading("produto", text="Insumo / Ingrediente")
        self.tabela.heading("quantidade", text="Qtd. Atual")
        self.tabela.heading("unidade", text="Unidade")
        self.tabela.heading("minimo", text="Estoque Mín.")
        self.tabela.heading("validade", text="Data de Validade")

        self.tabela.column("id", width=40, anchor=tk.CENTER)
        self.tabela.column("produto", width=220)
        self.tabela.column("quantidade", width=80, anchor=tk.CENTER)
        self.tabela.column("unidade", width=70, anchor=tk.CENTER)
        self.tabela.column("minimo", width=90, anchor=tk.CENTER)
        self.tabela.column("validade", width=110, anchor=tk.CENTER)

        # Tags de cores para os alertas de tela
        self.tabela.tag_configure("vencido", background="#FFCDD2")
        self.tabela.tag_configure("alerta_validade", background="#FFF9C4")
        self.tabela.tag_configure("estoque_baixo", background="#FFE0B2")
        self.tabela.tag_configure("normal", background="white")

        self.tabela.pack(fill=tk.BOTH, expand=True, pady=5)
        self.tabela.bind("<<TreeviewSelect>>", self.carregar_campos)

    def salvar_produto(self):
        """Valida, insere ou atualiza o insumo no banco de dados."""
        prod = self.inputs["txt_produto"].get().strip()
        unidade = self.inputs["cb_unidade"].get().strip()
        val_str = self.inputs["txt_validade"].get().strip()
        
        try:
            qtd = float(self.inputs["txt_qtd"].get())
            txt_min = float(self.inputs["txt_min"].get())
            # Valida formato da data DD/MM/AAAA
            datetime.strptime(val_str, "%d/%m/%Y")
        except ValueError:
            messagebox.showerror("Erro de Preenchimento", "Verifique se:\n1. Quantidade e Mínimo são números.\n2. Validade segue o formato DD/MM/AAAA.")
            return

        if not prod or not unidade:
            messagebox.showerror("Erro", "Por favor, preencha o nome do insumo e a unidade.")
            return

        # Verifica se o produto já existe para atualizar, ou insere novo
        self.cursor.execute("SELECT id FROM estoque WHERE produto = ?", (prod,))
        existe = self.cursor.fetchone()

        if existe:
            self.cursor.execute("""
                UPDATE estoque SET quantidade=?, unidade=?, estoque_minimo=?, validade=? WHERE produto=?
            """, (qtd, unidade, txt_min, val_str, prod))
        else:
            self.cursor.execute("""
                INSERT INTO estoque (produto, quantidade, unidade, estoque_minimo, validade) VALUES (?, ?, ?, ?, ?)
            """, (prod, qtd, unidade, txt_min, val_str))

        self.conn.commit()
        self.limpar_campos()
        self.atualizar_tabela()
        messagebox.showinfo("Sucesso", f"O item '{prod}' foi registrado digitalmente!")

    def atualizar_tabela(self):
        """Busca os dados no SQLite e aplica a lógica de alertas visuais por cores."""
        for row in self.tabela.get_children():
            self.tabela.delete(row)

        self.cursor.execute("SELECT * FROM estoque ORDER BY produto ASC")
        hoje = datetime.now()

        for item in self.cursor.fetchall():
            id_item, prod, qtd, unidade, est_min, validade_str = item
            
            # Lógica de classificação de cores (Regras de negócio da Padaria)
            try:
                data_val = datetime.strptime(validade_str, "%d/%m/%Y")
                dias_restantes = (data_val - hoje).days
            except ValueError:
                dias_restantes = 999 

            if dias_restantes < 0:
                tag = "vencido"
            elif dias_restantes <= 7:
                tag = "alerta_validade"
            elif qtd <= est_min:
                tag = "estoque_baixo"
            else:
                tag = "normal"

            self.tabela.insert("", tk.END, values=item, tags=(tag,))

    def carregar_campos(self, event):
        """Preenche o formulário ao clicar em um item da tabela (facilita alteração)."""
        selecionado = self.tabela.selection()
        if not selecionado:
            return
        
        valores = self.tabela.item(selecionado[0], 'values')
        self.limpar_campos()
        self.inputs["txt_produto"].insert(0, valores[1])
        self.inputs["txt_qtd"].insert(0, valores[2])
        self.inputs["cb_unidade"].set(valores[3])
        self.inputs["txt_min"].insert(0, valores[4])
        self.inputs["txt_validade"].insert(0, valores[5])

    def remover_produto(self):
        """Exclui o item selecionado do sistema."""
        selecionado = self.tabela.selection()
        if not selecionado:
            messagebox.showwarning("Aviso", "Selecione um insumo na tabela para remover.")
            return

        valores = self.tabela.item(selecionado[0], 'values')
        if messagebox.askyesno("Confirmar Exclusão", f"Deseja deletar '{valores[1]}' do estoque?"):
            self.cursor.execute("DELETE FROM estoque WHERE id = ?", (valores[0],))
            self.conn.commit()
            self.limpar_campos()
            self.atualizar_tabela()

    def limpar_campos(self):
        for entry in self.inputs.values():
            if isinstance(entry, ttk.Combobox):
                entry.set('')
            else:
                entry.delete(0, tk.END)

if __name__ == "__main__":
    root = tk.Tk()
    app = SistemaEstoquePadaria(root)
    root.mainloop()
