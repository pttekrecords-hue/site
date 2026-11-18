# PT.TEK | Backlog de melhorias do site

Origem: "PT.TEK — Sugestões de melhoria do site".
Regras: seguir o `AGENTS.md`. Não publicar, não apagar dados, não inventar links, faixas ou credenciais.
Princípio: o site continua leve. Sem base de dados, sem cache de faixas, sem bibliotecas pesadas.

## Como editar este ficheiro

- Cada tarefa tem um ID `T-NNN`. Para acrescentar, usa o próximo número livre. Não reutilizar IDs antigos.
- O estado de cada tarefa fica na coluna "Estado": `por fazer`, `em curso`, `concluída`, `bloqueada`.
- Textos do site vivem em `content/` (ver secção "Conteúdo e assets"). Mudar um texto não exige mexer no código.

## Conteúdo e assets (edição fácil)

Estrutura proposta. Cada pasta tem um ficheiro de texto e um manifesto, para que acrescentar um artista ou uma faixa seja copiar ficheiros e editar uma linha.

```
content/
├── textos.md            textos gerais: manifesto, princípios, contacto
├── artistas.json        um objeto por artista (nome, género, cor, imagem, faixa)
└── principios.md        os 6 princípios da Generosity Party
assets/
├── audio/               uma faixa por artista, com o nome em minúsculas e sem espaços
├── img/artists/         retratos 1024×1024, escala de cinza
├── img/bg/              fundos 1920×1080 (opcionais)
└── fonts/               fontes locais (ver T-109)
```

Regras para acrescentar um asset:
1. Coloca o ficheiro na pasta certa, com o nome em minúsculas e sem espaços.
2. Acrescenta uma entrada em `artistas.json` com o caminho relativo.
3. Corre o verificador de caminhos (T-101). Se o caminho estiver partido, o verificador indica qual.
4. Nunca adiciones uma faixa sem confirmar direitos de uso. Faixas sem licença escrita ficam fora do site.

## Tarefas

| ID | Estado | Área | Tarefa | Critério de aceitação | Validação |
|---|---|---|---|---|---|
| T-101 | por fazer | Conteúdo | Verificador de caminhos para `content/` e `assets/` | Lista todos os caminhos partidos e falha se houver algum | Teste automático |
| T-102 | por fazer | Conteúdo | Separar textos, artistas e princípios em `content/` | Alterar um texto em `content/` muda o site sem tocar no código | Teste de regressão |
| T-103 | por fazer | Conteúdo | Integrar logo, retratos e faixas reais quando existirem | Ficheiros presentes; sem referências partidas | T-101 |
| T-104 | por fazer | UX | Estado vazio do player sem faixas ("Faixa em breve") | Nenhum botão sem ação visível | Teste de regressão |
| T-105 | por fazer | UX | Caminho principal visível: ouvir → artista → submeter | Os três passos acessíveis a partir do topo | Revisão visual desktop e mobile |
| T-106 | por fazer | Acessibilidade | Navegação por teclado no player e `aria-live` para faixa e BPM | Todos os controlos alcançáveis com Tab; anúncio ao mudar de faixa | axe-core |
| T-107 | por fazer | Acessibilidade | Limite de três flashes por segundo | Nenhum efeito excede o limite, mesmo com BPM acima de 180 | Teste unitário no `BeatAgent` |
| T-108 | por fazer | Acessibilidade | `prefers-reduced-motion` em todos os efeitos com pulso | Pulso substituído por variação de brilho | Teste de CSS e revisão manual |
| T-109 | por fazer | Performance | Fontes self-hosted em vez de Google Fonts | Nenhum pedido a domínios externos de fontes | Teste de rede |
| T-110 | por fazer | Performance | Imagens em WebP/AVIF com `srcset`, dimensões e `loading="lazy"` | CLS medido antes e depois e registado | Lighthouse |
| T-111 | por fazer | Performance | Áudio com `preload="none"` até ao gesto do utilizador | Nenhum pedido de áudio antes do primeiro clique | Inspeção de rede |
| T-112 | por fazer | Engenharia | `AudioContext` criado só após gesto do utilizador | Sem erros de autoplay em Chromium e WebKit | Teste de navegador |
| T-113 | por fazer | Engenharia | Content-Security-Policy e SRI nos scripts externos | Cabeçalho presente; SRI em cada script de CDN | Teste de cabeçalhos |
| T-114 | por fazer | Engenharia | Suite de regressão do áudio: nó único por elemento e pausa que para as reações | Testes verdes | `node --test tests/*.test.cjs` |
| T-115 | por fazer | SEO | Metadados, Open Graph, Twitter Cards e JSON-LD | Validador de dados estruturados sem erros | Teste de HTML |
| T-116 | por fazer | SEO | `sitemap.xml`, `robots.txt` e URL canónica | Ficheiros coerentes com o domínio | Teste automático |
| T-117 | por fazer | PWA | `manifest.json` e Service Worker com página offline | Instalável; página offline carrega sem rede | Verificação do manifest e do SW |
| T-118 | por fazer | PWA | Cache de áudio limitado e só após confirmação | Sem cache automático de faixas | Revisão de código |
| T-119 | por fazer | Textos | Revisão dos textos de botões, estados vazios e do convite de submissão | Textos aprovados pelo utilizador | Revisão humana |
| T-120 | por fazer | Legal | Política de privacidade e aviso de cookies, se houver cookies não essenciais | Publicado só após revisão | Revisão humana; sem aconselhamento jurídico |
| T-121 | por fazer | Fórum | Página de fórum para a comunidade, com integração leve (ver secção "Fórum") | Carrega só quando o utilizador abre a página; nada no caminho principal | Lighthouse e inspeção de rede |
| T-122 | por fazer | Fórum | Regras de conduta e moderação escritas antes de abrir o fórum | Regras visíveis na página do fórum; responsável de moderação indicado | Revisão humana |
| T-123 | por fazer | Fórum | Secção "Aprender a produzir" com guias curtos ligados ao fórum | Cada guia tem data, autor e estado de revisão | Revisão humana |

## Fórum da comunidade (aprender a produzir)

Objetivo: um espaço para integrantes da comunidade PT.TEK fazerem perguntas, partilharem progresso e aprenderem produção.

Restrições: o site é estático e deve continuar leve. Por isso, o fórum não tem base de dados própria nem servidor próprio.

Opções, da mais leve para a mais completa:

| Opção | Peso no site | Vantagens | Desvantagens |
|---|---|---|---|
| Giscus (GitHub Discussions) | Carrega só na página do fórum; zero servidor | Gratuito, moderação no GitHub, sem base de dados | Exige conta GitHub; dados tratados fora da UE; público técnico |
| Discourse alojado | Script externo mais pesado, carregado só no fórum | Fórum completo, moderação e notificações | Custo mensal; mais dados pessoais a gerir |
| Lista de email ou página de perguntas | Mínimo | Simples e sem terceiros | Pouca interação |

Recomendação: começar com Giscus, carregado de forma preguiçosa apenas na página do fórum. Antes de abrir, confirmar as regras de conduta (T-122) e a informação de privacidade sobre o tratamento de dados pelo GitHub (T-120).

Notas:
- Não criar contas nem credenciais em nome da comunidade sem autorização.
- Não recolher dados de menores sem consentimento. Se o fórum aceitar menores, a conversa com o responsável deve acontecer antes.
- Não faças cache de conteúdo do fórum no Service Worker.

## Ordem de execução

1. T-101, T-102: verificador e separação de conteúdo. Sem isto, as edições futuras ficam difíceis.
2. T-114, T-112: base de áudio e testes.
3. T-104 a T-108: UX e acessibilidade.
4. T-109 a T-111: performance e privacidade.
5. T-113, T-115, T-116, T-117, T-118: segurança, SEO e PWA.
6. T-119, T-120, T-122: textos, legal e regras do fórum, com revisão humana.
7. T-121, T-123: fórum e secção de aprendizagem, só depois dos passos 5 e 6.

## Entrega final

Para cada tarefa: ID, ficheiros alterados, teste antes e depois, resultado e riscos.
No fim: `python tools/diagnose.py` e o resumo do estado de todas as tarefas.
Testes de navegador só se declaram executados se tiverem sido executados.
