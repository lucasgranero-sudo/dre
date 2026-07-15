# DRE - Fechamento

Calculadora de DRE (fechamento de resultado) para vendedores do Mercado Livre.

Site estático de arquivo único: **não tem build, não tem servidor, não tem instalação**.
Abrir `index.html` no navegador já funciona.

## Como rodar localmente

Basta abrir o arquivo:

```bash
open index.html          # macOS
start index.html         # Windows
```

Se preferir servir por HTTP (opcional):

```bash
python3 -m http.server 8000
# depois acesse http://localhost:8000
```

## O que ele faz

1. **Upload único do relatório de vendas da ML** — preenche automaticamente faturamento,
   comissão e frete; lista as vendas com reclamação; extrai a quantidade vendida por produto.
2. **Planilha de produtos e custos** — cruza com as quantidades e calcula o CMV.
   Quem não tem a planilha pode gerar um modelo já preenchido com os próprios produtos.
3. **Preenchimento manual** do que não existe no relatório (Ads, Full, frete reverso,
   custos fixos) e cálculo de EBITDA, impostos, lucro líquido e margens.
4. **Salvar por cliente/competência** e exportar para Excel.

Tudo roda no navegador. Nenhum dado é enviado para servidor.

## Publicar

### Netlify (mais simples)

Arraste a pasta em https://app.netlify.com/drop — sai um link público na hora.

Para deploy contínuo (cada `git push` publica sozinho):

```bash
git init
git add .
git commit -m "DRE - Fechamento"
# crie o repositório no GitHub e envie
git remote add origin git@github.com:USUARIO/dre-fechamento.git
git push -u origin main
```

Depois, no Netlify: **Add new site → Import an existing project** → escolha o repositório.
Sem build command. Publish directory: `.` (raiz).

### GitHub Pages

Settings → Pages → Source: `main` / root. O site sai em
`https://USUARIO.github.io/dre-fechamento/`.

### Domínio próprio

Tanto Netlify quanto GitHub Pages aceitam domínio customizado com HTTPS gratuito
(ex.: `dre.seudominio.com.br`) apontando um CNAME no seu provedor de DNS.

## Importante

- Os fechamentos salvos ficam no **localStorage do navegador de cada usuário**.
  Trocar de computador ou limpar o navegador apaga os dados salvos.
  Para acesso multi-dispositivo seria necessário backend + login (mudança grande de escopo).
- Antes de mudar qualquer regra de cálculo, leia o `CLAUDE.md` — ele documenta as decisões
  já tomadas e validadas com o cliente.
