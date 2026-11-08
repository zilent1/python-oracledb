# Release do `oracledb-o5logon` — passo a passo do que falta

Estado em 2026-11-08:

- Branch `release/o5logon` no fork (`zilent1/python-oracledb`), em `d30b3d29`:
  - `3eec7b01` patch O5LOGON 10G (rebaseado sobre `v26.1.0`)
  - `f1ddc428` TripleDES com chave de 24 bytes
  - `c6a29f3f` empacotamento: `oracledb-o5logon` 26.1.0.1, workflow, `.gitmodules`
  - `d30b3d29` README com a seção do verifier 10G
- Verificado **offline**: build local, `twine check`, wheel instalado em venv
  limpo, verifier 10G batendo com hashes conhecidos.
- **Não** verificado: handshake O5LOGON contra banco (passo 1), build no GitHub
  Actions, publicação.
- Nome `oracledb-o5logon` livre no PyPI e no TestPyPI (404 em 2026-11-08).

Ordem: 1 → 2 → 3 → 4 → 5 → 6. O passo 1 (banco) é o único que pode mudar
código; se ele falhar, tudo depois espera. O 1b (19c real) é opcional.

---

## 1. Testar o handshake no Oracle XE 21c (Docker)

Banco descartável, local, sem login na Oracle. Usa a imagem de comunidade
`gvenzl/oracle-xe:21-slim` (21c XE, ~2 GB).

**Por que 21c e não a imagem Free (23ai/26ai):** o 23ai removeu o verifier
10G, então lá o teste não tem como passar. O 11g XE também não serve, porque o
thin mode exige 12.1+. O 21c é a versão mais nova que ainda aceita 10G e a
mais próxima do 19c de produção.

O fluxo cobre quatro coisas:

| # | Verificação | Prova |
|---|---|---|
| 1.4 | `test_9900` passa | o fork autentica uma conta só-10G criada pelo próprio teste |
| 1.5 | prova manual | a conta é mesmo só-10G (`PASSWORD_VERSIONS = 10G`), a conexão é thin, e o login ignora caixa como o 10G exige |
| 1.6 | contraprova | o `oracledb` oficial 26.1.0 falha com DPY-3015 na mesma conta |
| 1.7 | regressão 12C | contas comuns (verifier 12C) continuam autenticando em thin mode |

### 1.1 Subir o container

```bash
docker run -d --name oxe21 \
  -p 127.0.0.1:1521:1521 \
  -e ORACLE_PASSWORD=Admin_12345 \
  gvenzl/oracle-xe:21-slim

# a imagem NÃO tem healthcheck: espera a linha de pronto no log
# (~1 min depois do download)
until docker logs oxe21 2>&1 | grep -q "DATABASE IS READY TO USE!"; do
  sleep 10; echo aguardando...
done
```

- A porta fica presa ao `127.0.0.1`, então o banco não aparece na rede.
- `ORACLE_PASSWORD` vale para SYS e SYSTEM.
- O banco de teste é o PDB `XEPDB1`.
- Um `ORA-06512 at "SYS.DBMS_QOPATCH"` no log da subida é ruído do XE, não
  impede nada.
- A `slim` não tem Oracle Text. Basta para os passos 1.4–1.6; o 1.7 precisa
  do schema completo da suíte e, portanto, da `gvenzl/oracle-xe:21-full`.

### 1.2 Liberar logon 10G no servidor

O padrão do 21c é `SQLNET.ALLOWED_LOGON_VERSION_SERVER=12`, que recusa o 10G
com ORA-01017 (não ORA-28040, ver 1.4). Troque para 11 dentro do container:

```bash
docker exec oxe21 bash -c '
  f=$($ORACLE_HOME/bin/orabasehome)/network/admin/sqlnet.ora
  sed -i "/ALLOWED_LOGON_VERSION_SERVER/d" "$f"
  echo "SQLNET.ALLOWED_LOGON_VERSION_SERVER=11" >> "$f"
  echo "== $f"; cat "$f"'
```

Vale para conexões novas, sem restart: cada conexão dedicada lê o
`sqlnet.ora` ao nascer. O `sed` apaga uma linha anterior do mesmo parâmetro,
para não ficar duplicado.

O 21c usa Oracle Home somente leitura: o `$ORACLE_HOME/network/admin` existe
mas é ignorado, e a config de rede fica em `orabasehome`
(`/opt/oracle/homes/OraDBHome21cXE/network/admin`). Escrever no lugar errado
não dá erro nenhum, só não tem efeito.

### 1.3 Ambiente do teste

```bash
cd ~/Development/python-oracledb
git switch release/o5logon
git submodule update --init
.venv/bin/pip install -q -e . pytest numpy pandas pyarrow   # recompila o Cython se mudou

export PYO_TEST_CONNECT_STRING="localhost:1521/XEPDB1"
export PYO_TEST_ADMIN_USER="system"
export PYO_TEST_ADMIN_PASSWORD="Admin_12345"
export PYO_TEST_MAIN_PASSWORD="Main_12345"     # obrigatória no conftest; o 9900 não usa
export PYO_TEST_PROXY_PASSWORD="Proxy_12345"   # só usada no 1.7
```

O `test_env.get_connection()` do conftest conecta primeiro com o usuário
principal (`pythontest`) para descobrir versão e charset. Sem ele, o 9900
falha com ORA-01017 **antes** de tentar o usuário 10G. Na `slim` o
`tests/create_schema.py` quebra (`ORA-01919: role 'CTXAPP'`), então crie só o
usuário:

```bash
docker exec oxe21 bash -c 'printf "create user pythontest identified by Main_12345;\ngrant create session to pythontest;\n" \
  | sqlplus -s -L system/Admin_12345@localhost/XEPDB1'
```

Antes de seguir, confirme que o `.venv` está com o build do fork:

```bash
.venv/bin/python -I -c "import oracledb; print(oracledb.__version__)"   # 26.1.0.1
```

### 1.4 `test_9900`

```bash
.venv/bin/python -m pytest tests/test_9900_verifier_10g.py -v -rs
```

Resultado esperado: **`1 passed`**. Um *skip* não conta: o `-rs` mostra o
motivo.

| Motivo do skip/erro | Causa | Ação |
|---|---|---|
| `missing administrative credentials` | `PYO_TEST_ADMIN_*` não exportadas | revisar 1.3 |
| `cannot provision a 10G-verifier account` | `IDENTIFIED BY VALUES` recusado | ver o ORA na mensagem |
| `ORA-01017` vindo de `conftest.py` (`_initialize`) | usuário `pythontest` não existe | criar como no 1.3 |
| `ORA-01017` vindo de `test_9900_verifier_10g.py:100` | **ou** o 1.2 não pegou, **ou** bug no handshake | tirar a dúvida com o `sqlplus` do container (abaixo) |
| `database does not allow 10G password verifiers` | ORA-28040; no 21c não aparece na prática (ver nota) | revisar 1.2 |

Com `ALLOWED_LOGON_VERSION_SERVER=12` o 21c responde **ORA-01017**, não
ORA-28040, para conta só-10G, e para todos os clientes (`sqlplus`, oficial,
fork). Por isso o teste falha em vez de pular num banco com config padrão.
Para separar config de bug:

```bash
docker exec oxe21 bash -c 'echo "select user from dual;" | sqlplus -s -L pyo_manual_v10g/Welcome_12345@localhost/XEPDB1'
```

(com a conta do 1.5 criada). `sqlplus` entra e o fork não → bug no patch.
Os dois falham → config do servidor.
| `DPY-3015` (falha) | o pytest está usando um `oracledb` sem o patch | conferir 1.3 (versão 26.1.0.1) |

### 1.5 Prova manual

O `test_9900` apaga o usuário no fim; este passo cria um usuário que fica até
o 1.8, para ser inspecionado e usado na contraprova.

Os scripts ficam fora do repo:

```bash
mkdir -p /tmp/o5 && cd /tmp/o5

cat > provision.py <<'EOF'
"""create|drop: conta PYO_MANUAL_V10G só com verifier 10G."""
import sys
import oracledb
from oracledb.thin_impl import get_verifier_10g   # só existe no fork

DSN, U, P = "localhost:1521/XEPDB1", "PYO_MANUAL_V10G", "Welcome_12345"
with oracledb.connect(user="system", password="Admin_12345", dsn=DSN) as conn:
    cur = conn.cursor()
    try:
        cur.execute(f"drop user {U} cascade")
    except oracledb.DatabaseError:
        pass
    if sys.argv[1] == "create":
        h = get_verifier_10g(U.encode(), P.encode()).hex().upper()
        cur.execute(f"create user {U} identified by values '{h}'")
        cur.execute(f"grant create session to {U}")
        cur.execute("select password_versions from dba_users where username = :u", u=U)
        print("hash 10G:", h, "| password_versions:", repr(cur.fetchone()[0]))
    print("ok:", sys.argv[1])
EOF

cat > connect.py <<'EOF'
"""Conecta como PYO_MANUAL_V10G com cada variação de caixa da senha."""
import oracledb

DSN = "localhost:1521/XEPDB1"
print("oracledb", oracledb.__version__)
for pw in ("Welcome_12345", "WELCOME_12345", "welcome_12345"):
    try:
        with oracledb.connect(user="pyo_manual_v10g", password=pw, dsn=DSN) as c:
            (u,) = c.cursor().execute("select user from dual").fetchone()
            print(f"  {pw}: OK  thin={c.thin} user={u}")
    except oracledb.Error as e:
        print(f"  {pw}: ERRO {e}")
EOF

~/Development/python-oracledb/.venv/bin/python -I provision.py create
~/Development/python-oracledb/.venv/bin/python -I connect.py
```

Esperado:

```
hash 10G: A16859259E315C8F | password_versions: '10G '      ← só 10G, nenhum 11G/12C
ok: create
oracledb 26.1.0.1
  Welcome_12345: OK  thin=True user=PYO_MANUAL_V10G
  WELCOME_12345: OK  thin=True user=PYO_MANUAL_V10G
  welcome_12345: OK  thin=True user=PYO_MANUAL_V10G
```

As três caixas passam porque o hash 10G põe usuário e senha em maiúsculas
antes de calcular. Se só a primeira passar, o fork não está aplicando o
`upper()` como o servidor; se nenhuma passar, ver a linha `ORA-01017` da
tabela do 1.4. Se `password_versions` vier com `11G` ou `12C`, a conta não é
só-10G e a prova não vale.

### 1.6 Contraprova com o driver oficial

```bash
python -m venv /tmp/o5/oficial
/tmp/o5/oficial/bin/pip install -q oracledb==26.1.0
cd /tmp/o5 && /tmp/o5/oficial/bin/python -I connect.py
```

Esperado:

```
oracledb 26.1.0
  Welcome_12345: ERRO DPY-3015: password verifier type 0x939 is not supported by python-oracledb in thin mode
  ...
```

As três tentativas devem dar DPY-3015. Isso mostra que a conta é a mesma, que
o servidor oferece o verifier 0x939 e que a diferença está só no fork.

### 1.7 Regressão: contas 12C (opcional, ~10 min)

O SYSTEM e o `pythontest` já autenticaram em thin mode nos passos
anteriores, e os dois têm verifier 12C. Para uma checagem mais ampla, rode a
parte de conexão da suíte do upstream. **Exige `gvenzl/oracle-xe:21-full`**
no 1.1 (a `slim` não tem a role `CTXAPP` que o `create_schema.py` usa); com a
`full`, pule a criação manual do `pythontest` no 1.3:

```bash
cd ~/Development/python-oracledb
.venv/bin/python -m pytest tests/create_schema.py -q          # cria pythontest/pythontestproxy
.venv/bin/python -m pytest tests/connection -q -rs -p no:cacheprovider
.venv/bin/python -m pytest tests/drop_schema.py -q
```

A suíte do upstream mira o 23ai/26ai. No 21c, testes de recurso novo (por
exemplo vetores, JSON nativo, sessionless transactions) falham ou são
pulados, com ou sem o fork. O que interessa aqui é **nenhuma falha de
autenticação** (ORA-01017, DPY-3015, DPY-4011 no connect). Para separar o que é
do fork do que é do 21c, rode a mesma linha com o `oracledb` oficial: as
falhas devem ser as mesmas.

### 1.8 Limpar

```bash
cd /tmp/o5 && ~/Development/python-oracledb/.venv/bin/python -I provision.py drop
docker rm -f oxe21
rm -rf /tmp/o5
# docker rmi gvenzl/oracle-xe:21-slim     # se não for usar de novo
```

Nada fica persistido: o container não tem volume, então `docker rm` apaga o
banco inteiro.

---

## 1b. Confirmar no 19c real (opcional)

O 21c cobre o protocolo. O 19c é onde o fork vai rodar; vale uma passada
antes do primeiro release se houver acesso a um banco não produtivo.

O `test_9900` cria um usuário só com verifier 10G (`IDENTIFIED BY VALUES`),
conecta com ele em thin mode e apaga o usuário no fim.

### 1b.1 Liberar logon 10G no servidor

Com o valor padrão do 19c (`SQLNET.ALLOWED_LOGON_VERSION_SERVER=12`) o
servidor recusa a conta só-10G. No 21c o erro é ORA-01017 para qualquer
cliente (o 19c deve se comportar igual), e o teste **falha** em vez de pular;
ver a nota do 1.4.

No servidor, em `$ORACLE_HOME/network/admin/sqlnet.ora` (ou no `$TNS_ADMIN`
em uso):

```
SQLNET.ALLOWED_LOGON_VERSION_SERVER=11
```

Vale para conexões novas, sem restart. Anote o valor anterior para voltar no
passo 1b.6.

### 1b.2 Ambiente de teste local

```bash
cd ~/Development/python-oracledb
git switch release/o5logon
git submodule update --init
.venv/bin/pip install -e . pytest numpy pandas pyarrow

export PYO_TEST_CONNECT_STRING="host:1521/servico"
export PYO_TEST_ADMIN_USER="system"          # precisa de CREATE/DROP USER
export PYO_TEST_ADMIN_PASSWORD="..."
export PYO_TEST_MAIN_PASSWORD="x"            # obrigatória no conftest; o 9900 não usa
```

O 9900 precisa que o usuário principal (`PYO_TEST_MAIN_USER`, padrão
`pythontest`) consiga conectar: o conftest conecta com ele antes de tudo. Use
um usuário de teste existente ou crie um só com `CREATE SESSION`, como no 1.3.

### 1b.3 Rodar

```bash
.venv/bin/python -m pytest tests/test_9900_verifier_10g.py -v -rs
```

Resultado esperado: `PASSED`. Atenção ao `-rs`: o teste **pula** (não falha)
em três casos, e pulado não conta como verificado:

| Motivo do skip | Causa | Ação |
|---|---|---|
| `missing administrative credentials` | `PYO_TEST_ADMIN_*` não exportadas | revisar 1b.2 |
| `cannot provision a 10G-verifier account` | `IDENTIFIED BY VALUES` recusado | ver mensagem ORA; checar privilégios do admin |
| `database does not allow 10G password verifiers` | ORA-28040 | revisar 1b.1 |

### 1b.4 Prova manual (opcional, recomendada)

Confirma que a conta é mesmo só-10G e que a conexão foi em thin mode. Em uma
sessão admin:

```sql
CREATE USER pyo_manual_v10g IDENTIFIED BY VALUES '<hash>';  -- hash do 9900
GRANT CREATE SESSION TO pyo_manual_v10g;
SELECT username, password_versions FROM dba_users
 WHERE username = 'PYO_MANUAL_V10G';                         -- esperado: "10G "
```

O hash sai de:

```bash
.venv/bin/python -I -c "from oracledb.thin_impl import get_verifier_10g as g; \
print(g(b'PYO_MANUAL_V10G', b'Welcome_12345').hex().upper())"
```

Depois:

```bash
.venv/bin/python -I -c "
import oracledb
c = oracledb.connect(user='pyo_manual_v10g', password='Welcome_12345',
                     dsn='host:1521/servico')
print('thin =', c.thin, '| user =', c.cursor().execute('select user from dual').fetchone())
"
```

Esperado: `thin = True | user = ('PYO_MANUAL_V10G',)`.

### 1b.5 Contraprova com o driver oficial

Mostra que o fork é o que resolve. Em outro venv:

```bash
python -m venv /tmp/oficial && /tmp/oficial/bin/pip install -q oracledb==26.1.0
/tmp/oficial/bin/python -I -c "
import oracledb
oracledb.connect(user='pyo_manual_v10g', password='Welcome_12345', dsn='host:1521/servico')
"
```

Esperado: `DPY-3015` (verifier type 0x939 não suportado).

### 1b.6 Limpar

```sql
DROP USER pyo_manual_v10g CASCADE;
```

E voltar `SQLNET.ALLOWED_LOGON_VERSION_SERVER` ao valor anotado em 1b.1, se
o banco não precisar de logons 10G por outro motivo.

**Se o passo 1b falhar:** corrigir no branch, novo commit, `git push`, e
repetir o passo 1 e o 1b antes de seguir.

---

## 2. Ativar o GitHub Actions no fork

Forks vêm com workflows desativados.

Pela web: aba **Actions** do fork → "I understand my workflows, go ahead and
enable them".

Ou pelo `gh`:

```bash
gh api -X PUT repos/zilent1/python-oracledb/actions/permissions \
  -F enabled=true -f allowed_actions=all
gh workflow enable build.yaml -R zilent1/python-oracledb
```

O fork também herda o `macaron-check-github-actions.yml`, que roda toda
quarta-feira no `main`. Não serve para o fork; desligue:

```bash
gh workflow disable macaron-check-github-actions.yml -R zilent1/python-oracledb
```

---

## 3. Criar os environments `pypi` e `testpypi`

O job `publish` usa `environment: pypi` em tag e `environment: testpypi` em
execução manual. O nome precisa bater exatamente com o cadastrado no PyPI
(passo 4).

### 3.1 `pypi`: só tags `o5-v*`, com sua aprovação

```bash
MYID=$(gh api user --jq .id)

gh api -X PUT repos/zilent1/python-oracledb/environments/pypi --input - <<EOF
{
  "reviewers": [{"type": "User", "id": $MYID}],
  "deployment_branch_policy": {"protected_branches": false,
                               "custom_branch_policies": true}
}
EOF

gh api -X POST repos/zilent1/python-oracledb/environments/pypi/deployment-branch-policies \
  -f name='o5-v*' -f type=tag
```

Com isso, cada release para no job `publish` esperando você clicar
**Approve** na execução. Uma tag criada por engano não vira release sozinha.

### 3.2 `testpypi`: só o branch `release/o5logon`, sem aprovação

```bash
gh api -X PUT repos/zilent1/python-oracledb/environments/testpypi --input - <<EOF
{
  "deployment_branch_policy": {"protected_branches": false,
                               "custom_branch_policies": true}
}
EOF

gh api -X POST repos/zilent1/python-oracledb/environments/testpypi/deployment-branch-policies \
  -f name='release/o5logon' -f type=branch
```

Conferir: Settings → Environments deve mostrar os dois.

---

## 4. Cadastrar o Trusted Publisher (PyPI e TestPyPI)

Sem token: o PyPI confia no OIDC do GitHub para esse repo + workflow +
environment. Como o projeto ainda não existe, é um **pending publisher**; ele
vira o projeto no primeiro upload.

Pré-requisito: conta com 2FA ativado em cada site. **PyPI e TestPyPI são
contas separadas.**

### 4.1 TestPyPI

https://test.pypi.org/manage/account/publishing/ → "Add a new pending
publisher" → GitHub:

| Campo | Valor |
|---|---|
| PyPI Project Name | `oracledb-o5logon` |
| Owner | `zilent1` |
| Repository name | `python-oracledb` |
| Workflow name | `build.yaml` |
| Environment name | `testpypi` |

### 4.2 PyPI

https://pypi.org/manage/account/publishing/ → mesmos campos, com
**Environment name = `pypi`**.

Um pending publisher reserva o nome só depois do primeiro upload. Até lá
outra pessoa pode registrar `oracledb-o5logon`, então não deixe muito tempo
entre este passo e o passo 6.

---

## 5. Ensaio completo no TestPyPI

### 5.1 Disparar

```bash
gh workflow run build.yaml -R zilent1/python-oracledb \
  --ref release/o5logon -f publish=true
gh run watch -R zilent1/python-oracledb \
  $(gh run list -R zilent1/python-oracledb -w build.yaml -L1 --json databaseId --jq '.[0].databaseId')
```

O `publish` só roda com `tgt` no valor padrão (`Linux, macOS, Windows,
Source`); não passe `-f tgt=...` no ensaio.

**Se o `gh` reclamar de input inesperado (`publish`):** o `main` do fork
ainda tem o `build.yaml` antigo do upstream. Rode sem `-f publish=true`
para validar só o build. Para o ensaio de upload, troque o default branch do
fork para `release/o5logon` temporariamente (Settings → General → Default
branch) e volte depois.

### 5.2 O que conferir na execução

- Os 4 jobs de build passaram. São cerca de 30 jobs (Linux 4 imagens, macOS
  6 Pythons, Windows 2 arquiteturas × 6 + arm64 × 5).
- O checkout com `submodules: true` baixou o ODPI-C de `oracle/odpi`. Se
  falhar aqui, o `.gitmodules` voltou a ser relativo.
- O artefato `python-oracledb-wheels` tem um `.tar.gz` e os wheels
  `oracledb_o5logon-26.1.0.1-*.whl`, com nenhum `oracledb-26.1.0*` sobrando.
- O job `publish` subiu para `test.pypi.org`.

Se algum build de Python 3.15 falhar por ser pré-release no runner, tirar a
3.15 das matrizes é aceitável; o upstream publica 3.15, mas o fork não
precisa.

### 5.3 Instalar do TestPyPI num venv limpo

```bash
python -m venv /tmp/tp && /tmp/tp/bin/pip install -q \
  -i https://test.pypi.org/simple/ --extra-index-url https://pypi.org/simple/ \
  oracledb-o5logon==26.1.0.1
cd /tmp && /tmp/tp/bin/python -I -c "
import oracledb, importlib.metadata as m
from oracledb.thin_impl import get_verifier_10g
print(oracledb.__version__, m.version('oracledb-o5logon'))
assert get_verifier_10g(b'SYSTEM', b'MANAGER').hex().upper() == 'D4DF7931AB130E37'
print('ok')
"
```

(`--extra-index-url` é para trazer `cryptography` e `typing_extensions` do
PyPI real.)

Opcional: repetir o 1.5/1.6 com esse wheel (trocar o `.venv` do repo pelo
`/tmp/tp/bin/python` no `connect.py`), para garantir que o pacote publicado
autentica.

Conferir também a página https://test.pypi.org/project/oracledb-o5logon/: o
README renderizado, o aviso de fork no topo e os links Source/Issues
apontando para `zilent1`.

**O TestPyPI não aceita reenviar a mesma versão.** Para um segundo ensaio,
suba a versão para `26.1.0.2` num commit temporário que não vá para o
release, ou aceite que o 2º ensaio pula o upload.

---

## 6. Release no PyPI

### 6.1 Conferências finais

```bash
cd ~/Development/python-oracledb
git switch release/o5logon && git pull --ff-only
git status --short                     # sem mudanças rastreadas pendentes
grep __version__ src/oracledb/version.py   # 26.1.0.1
```

### 6.2 Tag

```bash
git tag -a o5-v26.1.0.1 -m "oracledb-o5logon 26.1.0.1 (python-oracledb 26.1.0 + O5LOGON 10G)"
git push origin o5-v26.1.0.1
```

O job `publish` confere que a tag (`o5-v` + versão) bate com o
`version.py` e falha antes do upload se não bater.

### 6.3 Aprovar

Os builds rodam (o mesmo tempo do ensaio) e o `publish` para em
"Waiting for review". Na página da execução: **Review deployments → pypi →
Approve and deploy**.

### 6.4 Verificar

```bash
python -m venv /tmp/rel && /tmp/rel/bin/pip install -q oracledb-o5logon==26.1.0.1
cd /tmp && /tmp/rel/bin/python -I -c "import oracledb; print(oracledb.__version__)"
```

E a página https://pypi.org/project/oracledb-o5logon/.

### 6.5 GitHub Release (opcional)

```bash
gh release create o5-v26.1.0.1 -R zilent1/python-oracledb \
  --title "oracledb-o5logon 26.1.0.1" \
  --notes "Based on python-oracledb 26.1.0. Adds thin mode O5LOGON support for accounts with only a 10G password verifier."
```

### 6.6 Se algo sair errado depois do upload

O PyPI nunca aceita reenviar a mesma versão, nem depois de apagar. Faça
**yank** da `26.1.0.1` (Manage project → Releases → Options → Yank),
corrija, suba para `26.1.0.2` e repita a partir de 6.1.

---

## Depois: acompanhar novos releases da Oracle

Quando a Oracle lançar, por exemplo, `v26.2.0`:

```bash
cd ~/Development/python-oracledb
git fetch upstream --tags
git switch release/o5logon
git rebase v26.2.0
```

- Conflito provável em `src/oracledb/version.py` (o commit de empacotamento
  muda a versão): resolva com `26.2.0.1`.
- Se o upstream mexer em `auth.pyx`/`crypto.pyx`, revisar o patch com
  atenção, mesmo sem conflito.
- Rodar de novo o passo 1 (XE 21c no Docker) e o passo 5 (ensaio TestPyPI).
- `git push --force-with-lease origin release/o5logon`, depois tag
  `o5-v26.2.0.1`.

O 4º componente volta a `.1` a cada nova base da Oracle e sobe (`.2`, `.3`)
para correções do fork na mesma base.
