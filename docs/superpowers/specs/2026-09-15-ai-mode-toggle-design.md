# Design: Toggle de IA (nuvem/local) no upload de DANFE

**Data:** 2026-09-15
**Status:** Aprovado

## Contexto

O backend (clever-invoice) agora aceita um campo `ai_mode` (`"cloud"` ou
`"local"`, padrão `"cloud"`) no `POST /api/v1/upload`, e retorna na resposta
`ai_provider_used` (`"cloud"`/`"local"`) e `ai_fallback_used` (bool),
indicando se a extração usou IA na nuvem (OpenAI) ou local (LM Studio), e se
houve fallback automático da nuvem para o local por indisponibilidade.

O frontend (`index.html`, single-page vanilla JS/CSS) precisa deixar o
usuário escolher o modo antes de enviar o arquivo, e mostrar no resultado
qual IA processou a nota.

## Requisitos

1. Um controle na tela de upload deixa o usuário escolher `cloud` ou `local`
   antes de clicar em "Processar DANFE".
2. A escolha é lembrada entre sessões via `localStorage` (chave `aiMode`),
   com fallback pro padrão `"cloud"` se não houver valor salvo ou se
   `localStorage` não estiver disponível.
3. O upload envia o campo `ai_mode` escolhido no `FormData`.
4. O resultado exibe um badge indicando qual IA processou a nota
   (`ai_provider_used`), sempre visível.
5. Se `ai_fallback_used === true`, um aviso adicional explica que a nuvem
   estava indisponível e a extração caiu para IA local.

## Design

### UI

Um segmented control com 2 botões (`Nuvem (OpenAI)` / `Local (LM Studio)`),
inserido dentro de `.upload-card`, acima de `.drop-zone`. Reaproveita o
padrão visual dos `.tab-btn` existentes (pill ativo com `--accent`), em uma
barra menor com `role="radiogroup"` e `aria-pressed` nos botões para
acessibilidade.

```html
<div class="ai-mode-toggle" role="radiogroup" aria-label="Modo de IA">
  <button type="button" class="ai-mode-btn" data-mode="cloud" aria-pressed="true">Nuvem (OpenAI)</button>
  <button type="button" class="ai-mode-btn" data-mode="local" aria-pressed="false">Local (LM Studio)</button>
</div>
```

CSS novo (`.ai-mode-toggle`, `.ai-mode-btn`, `.ai-mode-btn.active`) segue o
mesmo vocabulário de cores/tokens já definidos em `:root` (nada de cor nova).

### Estado e persistência

```js
let aiMode = 'cloud'
try {
  aiMode = localStorage.getItem('aiMode') || 'cloud'
} catch { /* localStorage indisponível (modo privado etc.) */ }
```

Clique em um botão do toggle: atualiza `aiMode`, alterna `.active`/
`aria-pressed` nos dois botões, e persiste:
```js
try { localStorage.setItem('aiMode', aiMode) } catch {}
```

### Upload

Em `btnUpload`'s click handler, uma linha adicional no `FormData` existente:
```js
form.append('ai_mode', aiMode)
```

### Resultado

Em `renderResult(data)`, popular um novo badge ao lado de `#resultBadge`
("Sucesso"), reaproveitando as classes `.badge`/`.badge-success` já
existentes: `"Processado via: Nuvem"` ou `"Processado via: Local"` conforme
`data.ai_provider_used`.

Se `data.ai_fallback_used === true`, exibir um `.alert.alert-warning`
(classe já estilizada no CSS, ainda não usada em lugar nenhum) acima da
seção de resultado, com texto explicando que a IA na nuvem estava
indisponível e a extração foi feita pela IA local. O aviso some/reseta a
cada novo `btnUpload` click e a cada `btnReset`.

### Erros

Sem mudanças — `showError` já exibe `data.detail`, e as mensagens 503 que o
backend retorna quando `ai_mode=local` falha já mencionam LM Studio
diretamente.

## Fora de escopo

- Detectar automaticamente se o LM Studio está rodando antes do upload
  (ex.: um "ping" prévio) — não pedido, adiciona complexidade sem
  necessidade imediata.
- Configurar a URL/modelo do LM Studio pela UI — isso é config de servidor
  (`.env` do backend), não faz sentido no frontend.
