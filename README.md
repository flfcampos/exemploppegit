
# Guia Completo: Versionamento de Soluções Power Platform com Git + GitHub Actions

---

## 0. Instalar ferramentas necessárias

### No macOS

1. Instale Homebrew (se não tiver):
```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

2. Instale .NET SDK:
```bash
brew install --cask dotnet-sdk
```

3. Adicione .NET ao PATH (se necessário):
```bash
export PATH="/usr/local/share/dotnet:$PATH"
```

4. Instale Power Platform CLI:
```bash
dotnet tool install -g Microsoft.PowerApps.CLI.Tool
```

5. Verifique Git (normalmente já vem instalado):
```bash
git --version
```

---

### No Windows

1. Baixe e instale o [.NET SDK](https://dotnet.microsoft.com/en-us/download).

2. Instale Power Platform CLI:  
- Opção 1: via instalador MSI oficial  
[Power Platform CLI MSI](https://aka.ms/PowerAppsCLI)  

- Opção 2: via PowerShell (após instalar .NET SDK)  
```powershell
dotnet tool install -g Microsoft.PowerApps.CLI.Tool
```

3. Instale Git ([Git para Windows](https://git-scm.com/download/win)).

---

## 1. Criar solução no Power Apps (ambiente teste)

- Crie e configure a solução no portal Power Apps com os apps e componentes necessários.
- Valide com usuários.

---

## 2. Autenticar no Power Platform via CLI

```bash
pac auth create --url https://<seu-ambiente>.crm.dynamics.com
```

---

## 3. Exportar a solução para arquivo ZIP

```bash
pac solution export --name NomeDaSolucao --path ./solucao.zip --managed false
```

---

## 4. Desempacotar a solução para pastas

```bash
pac solution unpack --zipfile ./solucao.zip --folder ./src --allowDelete true
```

---

## 5. Inicializar Git e enviar para GitHub

```bash
git init
git remote add origin https://github.com/seuusuario/seurepo.git
git add .
git commit -m "Versão inicial da solução"
git push -u origin main
```

---

## 6. Criar Azure App Registration para autenticação

1. Acesse o [Azure Portal](https://portal.azure.com).

2. Vá em **Azure Active Directory > Registros de aplicativos > Novo registro**.

3. Informe:  
- Nome: `PowerPlatformGitHub` (ou outro nome)  
- Suporte de conta: “Contas nesta organização”  
- URI de redirecionamento: deixe em branco

4. Clique em **Registrar**.

5. Vá em **API Permissions > Adicionar permissão**.

6. Selecione **APIs Microsoft > Common Data Service**.

7. Adicione permissão **Application** `user_impersonation`.

8. Clique em **Conceder consentimento do administrador**.

9. Vá em **Certificates & secrets > Novo segredo do cliente**.

10. Crie segredo e copie o valor.

11. Pegue os IDs do app e tenant na página inicial do App Registration.

---

## 7. Configurar segredos no GitHub

No seu repositório, configure os secrets:

| Nome do segredo   | Valor                                   |
|-------------------|----------------------------------------|
| `PP_CLIENT_ID`    | ID do aplicativo (cliente) do Azure AD |
| `PP_TENANT_ID`    | ID do diretório (tenant) do Azure AD   |
| `PP_CLIENT_SECRET`| Segredo criado no App Registration      |
| `PP_ENV_URL`      | URL do ambiente Power Platform          |

---

## 8. Criar workflow GitHub Actions

Crie o arquivo `.github/workflows/powerplatform.yml` com:

```yaml
name: Power Platform CI/CD

on:
  push:
    branches:
      - dev
      - test
      - prod

jobs:
  export-unpack:
    if: github.ref == 'refs/heads/dev'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Autenticar no Power Platform
        uses: microsoft/powerplatform-actions/authenticate@v0
        with:
          environment-url: ${{ secrets.PP_ENV_DEV_URL }}
          client-id: ${{ secrets.PP_CLIENT_ID }}
          client-secret: ${{ secrets.PP_CLIENT_SECRET }}
          tenant-id: ${{ secrets.PP_TENANT_ID }}

      - name: Exportar solução
        uses: microsoft/powerplatform-actions/export-solution@v0
        with:
          solution-name: NomeDaSolucao
          solution-output-file: ./solucao.zip

      - name: Desempacotar solução
        uses: microsoft/powerplatform-actions/unpack-solution@v0
        with:
          solution-zip-file: ./solucao.zip
          solution-folder: ./src
          allow-delete: true

      - name: Commit e push (opcional)
        run: |
          git config user.name "github-actions"
          git config user.email "actions@github.com"
          git add .
          git commit -m "Atualização automática da solução"
          git push origin dev
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  import-solution-test:
    if: github.ref == 'refs/heads/test'
    needs: export-unpack
    runs-on: ubuntu-latest
    environment: homologacao
    steps:
      - uses: actions/checkout@v3

      - name: Autenticar no Power Platform (Test)
        uses: microsoft/powerplatform-actions/authenticate@v0
        with:
          environment-url: ${{ secrets.PP_ENV_TEST_URL }}
          client-id: ${{ secrets.PP_CLIENT_ID }}
          client-secret: ${{ secrets.PP_CLIENT_SECRET }}
          tenant-id: ${{ secrets.PP_TENANT_ID }}

      - name: Importar solução Test
        uses: microsoft/powerplatform-actions/import-solution@v0
        with:
          solution-zip-file: ./solucao.zip
          publish-workflows: true

  import-solution-prod:
    if: github.ref == 'refs/heads/prod'
    needs: export-unpack
    runs-on: ubuntu-latest
    environment: producao
    steps:
      - uses: actions/checkout@v3

      - name: Autenticar no Power Platform (Prod)
        uses: microsoft/powerplatform-actions/authenticate@v0
        with:
          environment-url: ${{ secrets.PP_ENV_PROD_URL }}
          client-id: ${{ secrets.PP_CLIENT_ID }}
          client-secret: ${{ secrets.PP_CLIENT_SECRET }}
          tenant-id: ${{ secrets.PP_TENANT_ID }}

      - name: Importar solução Prod
        uses: microsoft/powerplatform-actions/import-solution@v0
        with:
          solution-zip-file: ./solucao.zip
          publish-workflows: true

```

---

## 9. Fluxo de trabalho 

- Desenvolva no Power Apps (Dev).  
- Faça commit e push no GitHub (branch main).  
- GitHub Actions exporta, desempacota, commita e importa solução automaticamente.

---

