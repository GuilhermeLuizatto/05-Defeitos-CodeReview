# 🔎 Formulário — Parte B

> Revisão estática de `usuarios.js` (~150 linhas) — sem execução do código.

**Dupla:** Guilherme Luizatto 222773 + Kauã Pimentel 223695
**Data da revisão:** 23/04/2026

---

### Finding #1

**📍 Linha(s):** 12

**🏷 Rótulo:** blocker

**📂 Dimensão:** Segurança

**⚠️ Severidade:** Crítica

**🐛 Problema:**
A função `buscarUsuarioPorNome` constrói a query SQL por concatenação direta do parâmetro `nome`, sem nenhuma sanitização. Isso abre uma vulnerabilidade clássica de SQL Injection: um atacante pode passar `' OR '1'='1` para retornar todos os registros, ou `'; DROP TABLE usuarios; --` para destruir dados. Não há prepared statement nem escape de caracteres especiais.

```javascript
// linha 12 — VULNERÁVEL
const query = "SELECT * FROM usuarios WHERE nome = '" + nome + "'";
```

**💡 Sugestão de correção:**
```javascript
// Usar query parametrizada (prepared statement)
async function buscarUsuarioPorNome(nome) {
  return db.executarQuery(
    'SELECT * FROM usuarios WHERE nome = ?',
    [nome]
  );
}
```

**📚 Referência:** OWASP Top 10 — A03:2021 Injection; CWE-89

---

### Finding #2

**📍 Linha(s):** 17–28

**🏷 Rótulo:** major

**📂 Dimensão:** Erros

**⚠️ Severidade:** Alta

**🐛 Problema:**
A constante `TIPOS_VALIDOS` é declarada na linha 5 e exportada, mas nunca usada internamente para validar `dados.tipo` antes de persistir o usuário. Qualquer string arbitrária — `"admin"`, `"superuser"`, `"root"` — pode ser inserida no banco, corrompendo a lógica de `calcularLimiteEmprestimo` (que faz `switch` por tipo) e abrindo brecha para escalada de privilégios.

```javascript
// linha 17 — TIPOS_VALIDOS existe mas nunca é consultado aqui
async function cadastrarUsuario(dados) {
  const usuario = {
    tipo: dados.tipo,  // qualquer string passa direto
    // ...
  };
}
```

**💡 Sugestão de correção:**
```javascript
async function cadastrarUsuario(dados) {
  if (!TIPOS_VALIDOS.includes(dados.tipo)) {
    throw new Error(
      `Tipo inválido: '${dados.tipo}'. Valores aceitos: ${TIPOS_VALIDOS.join(', ')}`
    );
  }
  // ... resto da função inalterado
}
```

---

### Finding #3

**📍 Linha(s):** 32–38

**🏷 Rótulo:** blocker

**📂 Dimensão:** Erros

**⚠️ Severidade:** Alta

**🐛 Problema:**
`atualizarEmail` não verifica se o usuário foi encontrado antes de acessar suas propriedades. Se `db.buscarPorId` retornar `null` (id inexistente), a linha `u.email = novoEmail` lança `TypeError: Cannot set properties of null`, expondo stack trace ao chamador e deixando a operação sem feedback claro para o cliente.

```javascript
// linha 32–34 — sem guard para null
async function atualizarEmail(id, novoEmail) {
  const u = await db.buscarPorId('usuarios', id);
  u.email = novoEmail;  // 💥 TypeError se u === null
```

**💡 Sugestão de correção:**
```javascript
async function atualizarEmail(id, novoEmail) {
  const u = await db.buscarPorId('usuarios', id);
  if (!u) {
    throw new Error(`Usuário com id '${id}' não encontrado.`);
  }
  u.email = novoEmail;
  await db.atualizar('usuarios', id, u);
  logger.info(`Email atualizado: userId=${id}`);
  return u;
}
```

---

### Finding #4

**📍 Linha(s):** 79–83

**🏷 Rótulo:** major

**📂 Dimensão:** Erros

**⚠️ Severidade:** Alta

**🐛 Problema:**
A verificação de `bloqueadoAte` ocorre **após** toda a árvore de decisão de limite, em vez de ser o primeiro passo da função. Isso viola o princípio fail-fast: se a lógica de cálculo ganhar efeitos colaterais futuros (logs, notificações, mutações de estado), eles serão executados mesmo para usuários que deveriam ser barrados imediatamente. Além disso, `calcularLimiteEmprestimo` ignora o campo `suspenso` para professores, enquanto `calcularLimiteComSuspensao` o verifica — comportamento divergente para o mesmo perfil de usuário.

```javascript
// linha 79 — bloqueio verificado só no final
if (usuario.bloqueadoAte && new Date(usuario.bloqueadoAte) > hoje) {
  return 0;
}
return limite;
```

**💡 Sugestão de correção:**
```javascript
function calcularLimiteEmprestimo(usuario) {
  const hoje = new Date();

  // fail-fast: verificações de bloqueio primeiro
  if (usuario.bloqueadoAte && new Date(usuario.bloqueadoAte) > hoje) return 0;
  if (usuario.suspenso) return 0;

  let limite = 5;
  // ... restante da lógica inalterado
}
```

---

### Finding #5

**📍 Linha(s):** 42–87 e 90–138

**🏷 Rótulo:** major

**📂 Dimensão:** Complexidade

**⚠️ Severidade:** Média

**🐛 Problema:**
`calcularLimiteEmprestimo` (linhas 42–87) possui **7 níveis de aninhamento** de `if/else`, com complexidade ciclomática estimada em >12 — acima do limiar recomendado de 10 (idealmente ≤5 por função). Toda a mesma tabela de limites está duplicada em `calcularLimiteComSuspensao` (linhas 90–138), violando o princípio DRY. Qualquer mudança de regra de negócio precisa ser replicada nos dois lugares, com alto risco de divergência silenciosa — já confirmada: as funções tratam `suspenso` de formas diferentes para professores.

```javascript
// Exemplo do aninhamento — 7 níveis de profundidade
if (usuario.tipo === 'professor') {
  if (usuario.tempoCasaEmDias > 365) {
    if (usuario.atrasos === 0) {
      // ...
        if (usuario.multaPendente) {  // nível 4+
```

**💡 Sugestão de correção:**
```javascript
// Extrair lógica compartilhada para uma função base
function _calcularLimiteBase(usuario) {
  if (usuario.tipo === 'professor') { /* ... */ }
  if (usuario.tipo === 'aluno')     { /* ... */ }
  return 5; // default
}

function calcularLimiteEmprestimo(usuario) {
  const hoje = new Date();
  if (usuario.bloqueadoAte && new Date(usuario.bloqueadoAte) > hoje) return 0;
  if (usuario.suspenso) return 0;
  return _calcularLimiteBase(usuario);
}

function calcularLimiteComSuspensao(usuario) {
  if (usuario.suspenso) return 0;
  return _calcularLimiteBase(usuario);
}
```

**📚 Referência:** Martin Fowler — *Refactoring* (Extract Function, Replace Nested Conditional with Guard Clauses)

---

### Finding #6

**📍 Linha(s):** 27 e 37

**🏷 Rótulo:** major

**📂 Dimensão:** Segurança

**⚠️ Severidade:** Média

**🐛 Problema:**
Endereços de e-mail — dados pessoais sob a LGPD (Lei 13.709/2018) — são escritos em texto claro nos logs de aplicação. Qualquer pessoa com acesso ao sistema de arquivos ou a um agregador de logs (Splunk, CloudWatch, Datadog) pode ler os e-mails dos usuários, violando o princípio da minimização de dados. Em caso de vazamento de logs, o impacto é direto sobre a privacidade dos titulares.

```javascript
// linha 27
logger.info('Usuario cadastrado: ' + dados.email);  // PII exposta

// linha 37
logger.info('Email atualizado: ' + novoEmail);       // PII exposta
```

**💡 Sugestão de correção:**
```javascript
// Logar apenas identificadores não-sensíveis
logger.info(`Usuario cadastrado: id=${usuario.id}`);
logger.info(`Email atualizado: userId=${id}`);
```

**📚 Referência:** LGPD Art. 6º — princípio da minimização; OWASP Logging Cheat Sheet

---

### Finding #7 *(extra)*

**📍 Linha(s):** 5 e 17–28

**🏷 Rótulo:** nit

**📂 Dimensão:** Padrões

**⚠️ Severidade:** Baixa

**🐛 Problema:**
`TIPOS_VALIDOS` é exportado como parte da API pública do módulo, mas é uma constante de controle interno de validação — não deveria ser um contrato externo. Exportá-la convida consumidores do módulo a depender dela para lógica própria, criando acoplamento desnecessário. Se os tipos válidos mudarem, todos os consumidores que importam a constante também precisam ser atualizados.

```javascript
// linha 5 + module.exports (última linha)
const TIPOS_VALIDOS = ['aluno', 'professor', 'funcionario', 'visitante'];
// ...
module.exports = { TIPOS_VALIDOS, ... };  // expõe detalhe interno
```

**💡 Sugestão de correção:**
```javascript
// Manter TIPOS_VALIDOS como constante privada do módulo
// e expor apenas as funções públicas
const TIPOS_VALIDOS = ['aluno', 'professor', 'funcionario', 'visitante']; // não exportar

module.exports = {
  listarUsuariosAtivos,
  buscarUsuarioPorNome,
  cadastrarUsuario,
  atualizarEmail,
  calcularLimiteEmprestimo,
  calcularLimiteComSuspensao
};
```

---

## ✅ Checklist final

- [x] Há pelo menos 6 findings preenchidas (7 ao total)
- [x] Cada finding cita linha, dimensão, rótulo e severidade
- [x] As sugestões são concretas e acionáveis (com código)
- [x] Pelo menos uma finding cobre segurança (Finding #1 e #6)
- [x] Pelo menos uma finding cobre complexidade (Finding #5)
