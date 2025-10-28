# advocacia
prazos e tarefas
import datetime
import json
import os

DATA_FILE = "tarefas.json"
ORDEM_PRIORIDADE = {"Urgente": 0, "Alta": 1, "Media": 2, "Baixa": 3}

def carregar_tarefas():
    if not os.path.exists(DATA_FILE):
        return []
    with open(DATA_FILE, "r", encoding="utf-8") as f:
        try:
            return json.load(f)
        except json.JSONDecodeError:
            print("Arquivo de dados corrompido. Iniciando com lista vazia.")
            return []

def salvar_tarefas(tarefas):
    with open(DATA_FILE, "w", encoding="utf-8") as f:
        json.dump(tarefas, f, ensure_ascii=False, indent=2)

def proximo_id(tarefas):
    if not tarefas:
        return 1
    return max(t.get("id", 0) for t in tarefas) + 1

def ler_data(prompt):
    while True:
        s = input(prompt).strip()
        try:
            dt = datetime.date.fromisoformat(s)
            return dt.isoformat()
        except ValueError:
            print("Formato inválido. Use AAAA-MM-DD.")

def ler_prioridade(prompt):
    while True:
        p = input(prompt).strip().capitalize()
        if p in ORDEM_PRIORIDADE:
            return p
        print("Prioridade inválida. Use: Urgente, Alta, Media ou Baixa.")

def adicionar_tarefa(tarefas):
    print("\n--- Adicionar Nova Tarefa ---")
    while True:
        descricao = input("Descrição da Tarefa: ").strip()
        if descricao:
            break
        print("Descrição não pode ficar vazia.")
    processo = input("Nº do Processo (ou ref.) [opcional]: ").strip()
    prazo_iso = ler_data("Prazo Final (AAAA-MM-DD): ")
    prioridade = ler_prioridade("Prioridade (Urgente/Alta/Media/Baixa): ")
    nova = {
        "id": proximo_id(tarefas),
        "descricao": descricao,
        "processo_num": processo,
        "prazo": prazo_iso,
        "prioridade": prioridade,
        "concluida": False,
        "criada_em": datetime.date.today().isoformat()
    }
    tarefas.append(nova)
    salvar_tarefas(tarefas)
    print(f"\n✅ Tarefa '{descricao}' adicionada (ID: {nova['id']}).")

def ordenar_tarefas_para_apresentacao(lista):
    def chave(t):
        try:
            prazo = datetime.date.fromisoformat(t.get("prazo"))
        except Exception:
            # se faltar/estiver inválido, coloca no final
            prazo = datetime.date.max
        prioridade_ord = ORDEM_PRIORIDADE.get(t.get("prioridade", ""), 99)
        return (prazo, prioridade_ord)
    return sorted(lista, key=chave)

def exibir_tarefas(tarefas, filtro="ativas"):
    if filtro == "ativas":
        subset = [t for t in tarefas if not t.get("concluida", False)]
        titulo = "Tarefas Ativas (ordenadas por prazo/prioridade)"
    elif filtro == "concluidas":
        subset = [t for t in tarefas if t.get("concluida", False)]
        titulo = "Tarefas Concluídas"
    else:
        subset = tarefas
        titulo = "Todas as Tarefas"

    if not subset:
        print("\n📝 Nenhuma tarefa encontrada para este filtro.")
        return

    ordenadas = ordenar_tarefas_para_apresentacao(subset)
    print(f"\n--- {titulo} ---")
    hoje = datetime.date.today()
    for t in ordenadas:
        prazo_dt = None
        status_prazo = ""
        try:
            prazo_dt = datetime.date.fromisoformat(t.get("prazo"))
            dias = (prazo_dt - hoje).days
            status_prazo = f"({dias} dias restantes)" if dias >= 0 else "(PRAZO VENCIDO!)"
        except Exception:
            status_prazo = "(Prazo inválido)"
        sinal = "X" if t.get("concluida") else " "
        print(f"ID {t['id']:>3} [{sinal}] [P:{t.get('prioridade','?')[:1]}] {t.get('descricao')}")
        proc = t.get("processo_num")
        if proc:
            print(f"    Processo: {proc}")
        print(f"    Prazo: {t.get('prazo')} {status_prazo}")
        if t.get("concluida") and t.get("concluida_em"):
            print(f"    Concluída em: {t.get('concluida_em')}")
        print("-" * 50)

def marcar_concluida(tarefas):
    exibir_tarefas(tarefas, filtro="ativas")
    ativas = [t for t in tarefas if not t.get("concluida", False)]
    if not ativas:
        return
    try:
        s = input("\nDigite o ID da tarefa para marcar como CONCLUÍDA: ").strip()
        id_selecionado = int(s)
    except ValueError:
        print("ID inválido.")
        return
    for t in tarefas:
        if t.get("id") == id_selecionado and not t.get("concluida", False):
            t["concluida"] = True
            t["concluida_em"] = datetime.date.today().isoformat()
            salvar_tarefas(tarefas)
            print(f"✅ Tarefa ID {id_selecionado} marcada como concluída.")
            return
    print("Nenhuma tarefa ativa encontrada com esse ID.")

def remover_tarefa(tarefas):
    exibir_tarefas(tarefas, filtro="todas")
    try:
        s = input("\nDigite o ID da tarefa para REMOVER (permanente): ").strip()
        id_sel = int(s)
    except ValueError:
        print("ID inválido.")
        return
    for idx, t in enumerate(tarefas):
        if t.get("id") == id_sel:
            confirma = input(f"Confirma remoção da tarefa '{t.get('descricao')}'? (s/N): ").strip().lower()
            if confirma == "s":
                tarefas.pop(idx)
                salvar_tarefas(tarefas)
                print("Tarefa removida.")
            else:
                print("Remoção cancelada.")
            return
    print("ID não encontrado.")

def menu_principal():
    tarefas = carregar_tarefas()

    # se não existir nenhum dado, sugerir tarefas de exemplo (apenas para teste)
    if not tarefas:
        tarefas.extend([
            {'id': 1, 'descricao': 'Analisar jurisprudência do TJ-SP para caso Z', 'prazo': '2025-10-25', 'processo_num': '0001', 'prioridade': 'Alta', 'concluida': False, 'criada_em': '2025-10-01'},
            {'id': 2, 'descricao': 'Protocolar defesa do cliente Silva - Prazo final!', 'prazo': '2025-10-21', 'processo_num': '0002', 'prioridade': 'Urgente', 'concluida': False, 'criada_em': '2025-10-02'},
            {'id': 3, 'descricao': 'Organizar arquivos físicos de 2024', 'prazo': '2026-01-30', 'processo_num': 'ADM-001', 'prioridade': 'Baixa', 'concluida': False, 'criada_em': '2025-09-01'},
        ])
        salvar_tarefas(tarefas)

    while True:
        print("\n" + "="*40)
        print("  📅 GESTÃO DE TAREFAS ADVOCACIA")
        print("="*40)
        print("1. Adicionar Tarefa")
        print("2. Visualizar Tarefas Ativas (Por Prazo)")
        print("3. Marcar Tarefa como Concluída")
        print("4. Visualizar Tarefas Concluídas")
        print("5. Remover Tarefa (permanente)")
        print("6. Visualizar Todas as Tarefas")
        print("7. Sair")
        escolha = input("Escolha uma opção: ").strip()
        if escolha == '1':
            adicionar_tarefa(tarefas)
        elif escolha == '2':
            exibir_tarefas(tarefas, filtro="ativas")
        elif escolha == '3':
            marcar_concluida(tarefas)
        elif escolha == '4':
            exibir_tarefas(tarefas, filtro="concluidas")
        elif escolha == '5':
            remover_tarefa(tarefas)
        elif escolha == '6':
            exibir_tarefas(tarefas, filtro="todas")
        elif escolha == '7':
            print("Saindo do sistema. Até breve!")
            break
        else:
            print("Opção inválida. Tente novamente.")

if __name__ == "__main__":
    menu_principal()
