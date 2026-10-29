# GitHub_action_test
Este repositório possui dois workflows do **GitHub Actions** que automatizam tarefas comuns no ciclo de desenvolvimento: a geração de um histórico de mudanças (changelog) e a publicação de releases.
 
| Workflow | Arquivo | Quando executa | O que faz |
|---|---|---|---|
| Changelog | `.github/workflows/changelog.yml` | Automaticamente, a cada push na `main` | Gera e atualiza o arquivo `CHANGELOG.md` |
| Release Package | `.github/workflows/release.yml` | Manualmente, quando o usuário dispara | Compacta o projeto em `.zip` e cria uma release no GitHub |
 
---
 
## 1. Workflow: Changelog
 
**Arquivo:** `changelog.yml`
 
### Objetivo
 
Manter um arquivo `CHANGELOG.md` sempre atualizado com o histórico de commits do projeto, sem que ninguém precise escrevê-lo à mão.
 
### Gatilho (`on`)
 
```yaml
on:
  push:
    branches: [ "main" ]
```
 
O workflow roda **automaticamente a cada push na branch `main`**.
 
### Permissões
 
```yaml
permissions:
  contents: write
```
 
Necessária porque o workflow precisa **escrever no repositório** (fazer commit e push do `CHANGELOG.md`).
 
### Passo a passo
 
1. **`actions/checkout@v4` com `fetch-depth: 0`**
   Baixa o código do repositório. O `fetch-depth: 0` faz o clone trazer **todo o histórico de commits** (por padrão, o checkout traz apenas o último commit, o que deixaria o changelog vazio).
2. **Gerar CHANGELOG.md**
   Executa um bloco de comandos shell que monta o arquivo:
   - escreve o título `# Changelog`;
   - adiciona a data e hora da geração (`date`);
   - roda `git log --pretty=format:"- %ad | %s (%an) [%h]" --date=short` para listar cada commit no formato:
```
     - 2025-01-15 | mensagem do commit (Autor) [abc1234]
```
 
     Onde `%ad` é a data, `%s` a mensagem, `%an` o autor e `%h` o hash curto do commit.
   - exibe o conteúdo no log com `cat` (útil para conferir na aba Actions).
3. **Enviar CHANGELOG.md para o repositório**
   - Configura o usuário do Git como `github-actions[bot]`;
   - Adiciona o `CHANGELOG.md` à área de stage;
   - Verifica com `git diff --cached --quiet` se houve alteração; se não houve, encerra sem fazer nada;
   - Caso haja mudanças, faz o commit `docs: update CHANGELOG.md [skip ci]` e o `git push`.
> O trecho `[skip ci]` na mensagem do commit indica ao GitHub que **não deve disparar novos workflows** por causa desse commit, evitando um loop infinito (o workflow commita, o que dispara o workflow, que commita de novo...).
 
### Resultado
 
Depois de cada push na `main`, o arquivo `CHANGELOG.md` na raiz do repositório é atualizado automaticamente com a lista completa de commits.
 
---
 
## 2. Workflow: Release Package
 
**Arquivo:** `release.yml`
 
### Objetivo
 
Empacotar o projeto em um arquivo `.zip` e publicá-lo como uma **Release** no GitHub, com uma versão definida pelo usuário.
 
### Gatilho (`on`)
 
```yaml
on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Número da versão'
        required: true
        default: 'v1.0.0'
```
 
O `workflow_dispatch` significa que o workflow é **executado manualmente**. Ao dispará-lo, o GitHub pede um campo de entrada:
 
- **`version`**: número da versão da release (obrigatório, com valor padrão `v1.0.0`).
### Permissões
 
```yaml
permissions:
  contents: write
```
 
Necessária para que o workflow consiga **criar a release e a tag** no repositório.
 
### Passo a passo
 
1. **`actions/checkout@v4`**
   Baixa o código do repositório na máquina virtual.
2. **Zip folder**
```bash
   git archive --format=zip -o release.zip HEAD
```
   Cria o arquivo `release.zip` com o conteúdo do commit atual (`HEAD`). O `git archive` inclui apenas os arquivos versionados, sem a pasta `.git`.
 
3. **Create GitHub Release**
```bash
   gh release create <versão> release.zip \
     --title "Release <versão>" \
     --generate-notes
```
   Usa o **GitHub CLI (`gh`)** para:
   - criar uma release (e a tag correspondente) com a versão informada;
   - anexar o `release.zip` como arquivo da release;
   - definir o título como `Release <versão>`;
   - gerar automaticamente as notas da release (`--generate-notes`), com base nos commits e pull requests desde a última release.
   A autenticação é feita pelo `GITHUB_TOKEN`, token temporário que o próprio GitHub fornece a cada execução.
 
### Como executar
 
1. Acesse a aba **Actions** do repositório.
2. Selecione o workflow **Release Package** na lista da esquerda.
3. Clique em **Run workflow**.
4. Informe a versão (por exemplo, `v1.0.1`) e confirme.
5. Ao terminar, a release aparecerá na seção **Releases** do repositório, com o `.zip` disponível para download.
> Cada versão só pode ser usada uma vez: se você tentar criar uma release com uma tag que já existe, o workflow falhará.
 
---
 
## Estrutura de arquivos
 
```
.
├── .github/
│   └── workflows/
│       ├── changelog.yml
│       └── release.yml
├── CHANGELOG.md        # gerado automaticamente
└── README.md
```
