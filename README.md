# Phython-ideas-and-etc
Py game and works etc sla








import sys

def run_flux(lines):
    vars = {}
    i = 0
    while i < len(lines):
        line = lines[i].strip()

        # Ignorar linhas vazias ou comentários
        if not line or line.startswith("#"):
            i += 1
            continue

        # print
        if line.startswith("print "):
            conteudo = line[6:].strip()
            try:
                print(eval(conteudo, vars))
            except:
                print(conteudo.strip('"').strip("'"))
            i += 1
            continue

        # variáveis
        if "=" in line and not line.startswith("if") and not line.startswith("while"):
            nome, valor = line.split("=", 1)
            nome = nome.strip()
            valor = valor.strip()
            try:
                vars[nome] = eval(valor, vars)
            except:
                vars[nome] = valor
            i += 1
            continue

        # if
        if line.startswith("if "):
            cond = line[3:].strip()[:-1]  # remove os dois pontos no final
            bloco = []
            i += 1
            while i < len(lines) and lines[i].startswith("    "):
                bloco.append(lines[i][4:])
                i += 1
            if eval(cond, vars):
                run_flux(bloco)
            else:
                # verificar se tem else logo em seguida
                if i < len(lines) and lines[i].strip().startswith("else:"):
                    i += 1
                    bloco_else = []
                    while i < len(lines) and lines[i].startswith("    "):
                        bloco_else.append(lines[i][4:])
                        i += 1
                    run_flux(bloco_else)
            continue

        # while
        if line.startswith("while "):
            cond = line[6:].strip()[:-1]
            bloco = []
            i += 1
            while i < len(lines) and lines[i].startswith("    "):
                bloco.append(lines[i][4:])
                i += 1
            while eval(cond, vars):
                run_flux(bloco)
            continue

        i += 1


if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("Uso: python flux.py arquivo.flux")
    else:
        with open(sys.argv[1], "r", encoding="utf-8") as f:
            run_flux(f.read().split("\n"))






            # Flux — Linguagem de Programação Simples e Rápida

Flux é uma linguagem de programação minimalista, com sintaxe fácil e foco em ser rápida para aprender e usar. Ideal para quem quer criar scripts simples e entender conceitos básicos de programação.

## Recursos

- Variáveis simples
- Impressão no terminal (`print`)
- Condicionais (`if` / `else`)
- Laços de repetição (`while`)
- Sintaxe clara e enxuta

## Como usar

1. Clone ou baixe este repositório.  
2. Certifique-se de ter Python instalado (versão 3.x).  
3. Rode seu código Flux com o interpretador:

```bash
python flux.py seu_codigo.flux




