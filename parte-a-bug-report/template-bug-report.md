# 🐛 Bug Reports — Parte A

**Dupla:** Guilherme Luizatto 222773 + Kauã Pimentel 223695
**Data da exploração:** 23/04/2026
**Navegador usado:** Chrome 121
**Sistema operacional:** Windows 10

---

## BUG-001

**Título:** Sistema aceita prazo com data no passado, em branco ou muito distante ao criar tarefa

**Severidade:** Alta
**Justificativa da severidade:** O sistema aceita datas inválidas sem qualquer validação. Tarefas com prazo no passado são criadas como pendentes, gerando inconsistência lógica. Datas em branco podem retornar null/undefined em cálculos de ordenação e filtros, causando comportamentos imprevisíveis na UI.

**Prioridade:** P1
**Justificativa da prioridade:** Quebra de ordenações por data; possível crash em funções de diff/comparação de datas; dados corrompidos persistem no backend; relatórios de produtividade ficam distorcidos; risco de NullPointerException ou erros de parse na API.

**Ambiente:**
- Navegador: Chrome 121.0
- Sistema Operacional: Windows 10
- Versão da aplicação: TarefaQS v1.0.0

**Passos para reprodução:**

*Cenário A — data no passado:*
1. Acessar a aplicação em guilhermeluizatto.github.io/05-Defeitos-CodeReview/parte-a-bug-report/app/index.html
2. No formulário "Nova tarefa", preencher o campo Título com "Tarefa passada"
3. Selecionar qualquer Categoria (ex: Estudo)
4. No campo Prazo, digitar ou selecionar uma data já encerrada (ex: 01/01/2020)
5. Preencher Prioridade com 3
6. Clicar em "Adicionar tarefa"
7. Observar que a tarefa é criada normalmente na lista, sem nenhum aviso

*Cenário B — prazo em branco:*
1. Repetir passos 1–3 do cenário A
2. Deixar o campo Prazo completamente vazio
3. Preencher os demais campos com valores válidos
4. Clicar em "Adicionar tarefa"
5. Observar que a tarefa é criada sem prazo, exibindo "Prioridade NaN" ou data inválida na lista

*Cenário C — data muito distante:*
1. Repetir passos 1–3 do cenário A
2. No campo Prazo, digitar 31/05/0004 (ano 4)
3. Clicar em "Adicionar tarefa"
4. Observar que a tarefa é criada sem erro

**Resultado esperado:**
- Cenário A: O sistema deve exibir mensagem de erro — "O prazo não pode ser anterior à data de hoje." — e bloquear a criação até que uma data futura válida seja informada.
- Cenário B: O campo Prazo deve ser tratado como obrigatório. Ao tentar salvar com prazo vazio, o sistema deve destacar o campo e exibir "Prazo é obrigatório." O botão "Adicionar tarefa" deve permanecer desabilitado ou a validação deve disparar no submit, impedindo a persistência.
- Cenário C: O sistema deve rejeitar anos fora de um intervalo razoável (ex: ano atual até +5 anos) e exibir aviso ao usuário.

**Resultado obtido:**
- Cenário A: A tarefa com prazo em 01/01/2020 é criada normalmente, aparece na lista como pendente e entra nos contadores, sem qualquer aviso ao usuário.
- Cenário B: A tarefa é criada com prazo vazio. Na listagem, o campo de data exibe "Prioridade NaN" (valor inválido vazou para a UI), evidenciando ausência de validação.
- Cenário C: A tarefa com prazo em 31/05/0004 é aceita e persistida. O campo data exibe o valor malformado na lista.

**Evidência:**
> Screenshot da lista exibindo tarefa com "Prioridade NaN" e data 31/05/0004 aceita sem erro.
> Arquivo: <img width="1919" height="1080" alt="image" src="https://github.com/user-attachments/assets/8b46c9f0-cd0f-41d4-a33b-34e6c1d813c6" />

**Sugestão de causa raiz:**
O campo de prazo é um `<input type="date">` sem atributo `min` definido (deveria ser `min="<data-de-hoje>"`). Não há validação JavaScript no evento de submit — o formulário não checa se o valor é vazio, passado ou absurdo antes de chamar a função de criação da tarefa. A exibição de "NaN" sugere que a lógica de renderização tenta calcular algo com a data (ex: `Date.parse(prazo)`) sem tratar o retorno `NaN` quando o campo está vazio.

---

## BUG-002

**Título:** Campo Prioridade aceita valores fora do domínio (0, 99, decimais, negativos) sem validação

**Severidade:** Crítica
**Justificativa da severidade:** O campo aceita inteiros arbitrários e floats sem restrição. Valores como 99 ou 3.5 não existem no enum de prioridades do domínio, corrompendo a lógica de negócio. Isso indica que não há validação nem no frontend nem (provavelmente) no backend, abrindo vetores para injeção de dados malformados.

**Prioridade:** P0
**Justificativa da prioridade:** Quebra de switches/condicionais baseados em enum de prioridade; falha silenciosa em sistemas de notificação e SLA; possível index out of range em arrays de labels; inconsistência entre dados do banco e lógica da aplicação; risco de exploração via API direta. Por ser uma falha arquitetural de validação dupla (front + back), dados inválidos persistem e podem propagar para sistemas integrados enquanto o bug existir.

**Ambiente:**
- Navegador: Chrome 121.0
- Sistema Operacional: Windows 10
- Versão da aplicação: TarefaQS v1.0.0

**Passos para reprodução:**

*Cenário A — valor acima do máximo:*
1. Acessar a aplicação
2. Preencher Título com "Tarefa prioridade 99"
3. Selecionar Categoria "Trabalho"
4. Preencher Prazo com uma data futura válida (ex: 30/06/2026)
5. No campo Prioridade (1–5), digitar **99**
6. Clicar em "Adicionar tarefa"
7. Observar a tarefa criada com "Prioridade 99" na listagem

*Cenário B — valor decimal:*
1. Repetir passos 1–4 do cenário A
2. No campo Prioridade, digitar **3.5**
3. Clicar em "Adicionar tarefa"
4. Observar que o valor é aceito

*Cenário C — valor zero:*
1. Repetir passos 1–4 do cenário A
2. No campo Prioridade, digitar **0**
3. Clicar em "Adicionar tarefa"
4. Observar que o valor 0 é aceito e exibido na listagem

**Resultado esperado:**
O campo Prioridade deve aceitar apenas inteiros entre 1 e 5 (conforme label "Prioridade (1-5)" exibida na UI). Para qualquer valor fora desse intervalo, o sistema deve:
- Exibir mensagem de validação inline: "Prioridade deve ser um número inteiro entre 1 e 5."
- Bloquear o submit até que o valor seja corrigido.
- Idealmente, implementar o campo como `<input type="number" min="1" max="5" step="1">` com validação JavaScript adicional, e validação espelhada no backend (retornando HTTP 400 para valores inválidos).

**Resultado obtido:**
Nos três cenários, a tarefa é criada normalmente com o valor inválido persistido. A listagem exibe "Prioridade 99", "Prioridade 3.5" e "Prioridade 0" sem nenhum aviso. O valor `7567567567567567567567` (observado na screenshot anexa ao template) também é aceito, confirmando ausência total de constraint no campo.

**Evidência:**
> Screenshot da listagem exibindo tarefas com "Prioridade 7567567567567567567567" e outros valores inválidos.
> Arquivo: <img width="1919" height="1080" alt="image" src="https://github.com/user-attachments/assets/8b46c9f0-cd0f-41d4-a33b-34e6c1d813c6" />
**Sugestão de causa raiz:**
O campo Prioridade é um `<input type="number">` sem atributos `min`, `max` ou `step` definidos. A função de criação de tarefas lê o valor diretamente com `parseInt()` ou `Number()` sem verificar se está no intervalo [1,5]. Não há validação no lado do servidor. A fix mínima é adicionar `min="1" max="5" step="1"` no HTML e uma guard clause no JavaScript antes de persistir: `if (prioridade < 1 || prioridade > 5 || !Number.isInteger(prioridade)) return exibirErro(...)`.

---

## BUG-003

**Título:** Tarefa criada com título vazio, muito curto, muito longo ou com emoji causa comportamentos inesperados na UI

**Severidade:** Alta
**Justificativa da severidade:** Sem preenchimento: campos obrigatórios não são validados; tarefa fantasma entra no sistema sem identificador legível. Título muito curto (1–2 chars): passa na criação mas quebra buscas e exibições; UI pode colapsar em layouts com ellipsis mínimo. Título muito longo: ausência de limite máximo pode causar overflow no banco (varchar), truncamento silencioso ou erro 500 na API. Emoji no título: caracteres multibyte (UTF-8 de 4 bytes) podem quebrar contagem de caracteres, truncamento incorreto e encoding em e-mails/integrações.

**Prioridade:** P1
**Justificativa da prioridade:** Registros órfãos no banco de dados; potencial DataTooLongException em colunas com limite; corrupção de encoding em exportações CSV/PDF; falha silenciosa em webhooks e integrações que consomem o campo título.

**Ambiente:**
- Navegador: Chrome 121.0
- Sistema Operacional: Windows 10
- Versão da aplicação: TarefaQS v1.0.0

**Passos para reprodução:**

*Cenário A — título vazio:*
1. Acessar a aplicação
2. Deixar o campo Título completamente em branco
3. Preencher Categoria, Prazo (futuro) e Prioridade (1–5) com valores válidos
4. Clicar em "Adicionar tarefa"
5. Observar que a tarefa é criada sem título na lista

*Cenário B — título com emoji:*
1. No campo Título, digitar três emojis seguidos: 😀😀😀
2. Preencher os demais campos com valores válidos
3. Clicar em "Adicionar tarefa"
4. Observar que os emojis aparecem quebrados ou mal renderizados na listagem

*Cenário C — título muito longo:*
1. No campo Título, colar uma string de 500+ caracteres (ex: "aaaa..." repetido)
2. Preencher os demais campos com valores válidos
3. Clicar em "Adicionar tarefa"
4. Observar o comportamento da UI e o valor persistido

*Cenário D — título com caracteres especiais (HTML):*
1. No campo Título, digitar: `<script>alert('xss')</script>`
2. Preencher os demais campos com valores válidos
3. Clicar em "Adicionar tarefa"
4. Observar se o script executa ou se o HTML é exibido como texto na lista

**Resultado esperado:**
- Cenário A: O campo Título deve ser obrigatório. Ao tentar salvar vazio, exibir "Título é obrigatório." e bloquear o submit.
- Cenário B: Emojis (caracteres Unicode multibyte) devem ser aceitos e exibidos corretamente. O sistema deve tratar a contagem de caracteres por codepoint Unicode, não por byte.
- Cenário C: O campo deve ter um limite máximo de caracteres (ex: 255). Ultrapassado o limite, o campo deve bloquear a digitação ou exibir erro. Nunca truncar silenciosamente no banco.
- Cenário D: Qualquer entrada HTML/script deve ser escapada e exibida como texto literal, nunca executada.

**Resultado obtido:**
- Cenário A: A tarefa é criada sem título. Na lista, aparece como uma entrada em branco (linha vazia), sem nenhum aviso ao usuário.
- Cenário B: Os emojis 😀😀😀 são exibidos incorretamente na listagem (caracteres quebrados ou substituídos por caixas). A string observada na screenshot (`\sfhadfhdghjdgshs dfgsr,nglçsDGHOÇSDFGVNÇSDKLSD/...`) confirma corrupção de encoding.
- Cenário C: Títulos com centenas de caracteres são aceitos integralmente, sem truncamento visível na UI e sem erro — o campo não tem `maxlength` definido.
- Cenário D: A ser verificado — ausência de sanitização nos outros cenários sugere risco de XSS reflexo.

**Evidência:**
> Screenshot da lista exibindo tarefa com título vazio e tarefa com encoding corrompido de emojis.
> Arquivo: <img width="1919" height="1080" alt="image" src="https://github.com/user-attachments/assets/8b46c9f0-cd0f-41d4-a33b-34e6c1d813c6" />

**Sugestão de causa raiz:**
O campo Título é um `<input type="text">` sem `required`, `minlength` ou `maxlength`. A função de criação não valida se `titulo.trim()` está vazio antes de persistir. O encoding corrompido dos emojis sugere que a renderização da lista usa manipulação direta de `innerHTML` ou `textContent` com string slicing baseado em `length` (que conta unidades UTF-16, não codepoints), quebrando pares substitutos de emojis ao truncar.

---

## ✅ Critérios de qualidade do bug report

- [x] Título descritivo — outra pessoa entende o problema só pelo título?
- [x] Passos são **numerados** e **reproduzíveis** por terceiros?
- [x] Há **pelo menos uma evidência** (screenshot, GIF ou log)?
- [x] Severidade tem **justificativa explícita**?
- [x] Prioridade tem **justificativa explícita**?
- [x] Ambiente inclui **navegador + SO**?
- [x] "Esperado vs. Obtido" deixa o gap claro?

## ✅ Checklist de qualidade dos reports

- [x] Título é específico e acionável
- [x] Passos estão **numerados** e são reproduzíveis por terceiros
- [x] Há **pelo menos uma evidência** por report (referência de arquivo)
- [x] Severidade tem **justificativa explícita**
- [x] Prioridade tem **justificativa explícita**
- [x] Ambiente inclui **navegador + SO**
- [x] "Esperado × Obtido" deixa a diferença clara
- [x] Os 3 defeitos cobrem **categorias diferentes**: validação de data (BUG-001), constraint de domínio numérico (BUG-002), validação de texto + encoding (BUG-003)
