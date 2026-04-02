# Tutorial: Oracle Instant Client + Zabbix Agent 2

Guia passo a passo para instalar e configurar o Oracle Instant Client com monitoramento via Zabbix Agent 2.

---

## 1. Pré-requisitos

- Oracle Instant Client >= 12
- Oracle Linux 6, 7 ou superior
- Acesso root ou sudo
- Conexão com o repositório Oracle e Zabbix

---

## 2. Instalação do Oracle Instant Client

### Para Oracle Linux 6 (OL6)

```bash
# Download e instalação do pacote
sudo yum install https://yum.oracle.com/repo/OracleLinux/OL6/oracle/instantclient/x86_64/getPackage/oracle-instantclient18.3-basic-18.3.0.0.0-3.x86_64.rpm

# Configurar biblioteca dinâmica
sudo sh -c "echo /usr/lib/oracle/18.3/client64/lib > /etc/ld.so.conf.d/oracle-instantclient.conf"

# Atualizar cache do loader
sudo ldconfig
```

### Para Oracle Linux 7+ (OL7+)

```bash
# Download e instalação do pacote
sudo yum install https://download.oracle.com/otn_software/linux/instantclient/1913000/oracle-instantclient19.23-basic-19.23.0.0.0-1.x86_64.rpm

# Configurar biblioteca dinâmica
sudo sh -c "echo /usr/lib/oracle/19.23/client64/lib > /etc/ld.so.conf.d/oracle-instantclient.conf"

# Atualizar cache do loader
sudo ldconfig
```

---

## 3. Configuração do sqlnet.ora

O arquivo `sqlnet.ora` controla comportamentos de rastreamento e diagnóstico do cliente Oracle. Desabilitaremos esses recursos para reduzir overhead.

### Para Oracle Linux 6 (OL6)

```bash
sudo cat > /usr/lib/oracle/18.3/client64/lib/network/admin/sqlnet.ora << 'EOF'
TRACE_TIMESTAMP_CLIENT=FALSE
TRACE_LEVEL_CLIENT=0
DIAG_ADR_ENABLED=OFF
EOF
```

### Para Oracle Linux 7+ (OL7+)

```bash
sudo cat > /usr/lib/oracle/19.23/client64/lib/network/admin/sqlnet.ora << 'EOF'
TRACE_TIMESTAMP_CLIENT=FALSE
TRACE_LEVEL_CLIENT=0
DIAG_ADR_ENABLED=OFF
EOF
```

**Nota:** Verifique se o diretório `/usr/lib/oracle/[versão]/client64/lib/network/admin/` existe antes de criar o arquivo.

---

## 4. Instalação do Zabbix Agent 2

### Adicionar repositório Zabbix

#### Para Oracle Linux 6 (OL6)

```bash
rpm -Uvh https://repo.zabbix.com/zabbix/7.0/rhel/6/x86_64/zabbix-release-latest-7.0.el6.noarch.rpm
```

#### Para Oracle Linux 7 (OL7)

```bash
rpm -Uvh https://repo.zabbix.com/zabbix/7.0/rhel/7/x86_64/zabbix-release-latest-7.0.el7.noarch.rpm
```

### Instalar o agent2

```bash
# Limpar cache
yum clean all

# Criar usuário do Zabbix
useradd -m -d /home/zabbix zabbix

# Instalar Zabbix Agent 2
yum install zabbix-agent2

# Criar diretório home para o usuário (se necessário)
mkhomedir_helper zabbix
```

---

## 5. Configuração do Zabbix Agent 2

### Editar arquivo de configuração

Abra o arquivo `/etc/zabbix/zabbix_agent2.conf` e configure os parâmetros do plugin Oracle:

```bash
sudo vi /etc/zabbix/zabbix_agent2.conf
```

### Parâmetros necessários

Procure ou adicione as seguintes linhas na seção do plugin Oracle:

```ini
Plugins.Oracle.CallTimeout=30
Plugins.Oracle.CustomQueriesPath=/home/zabbix/sql
Plugins.Oracle.KeepAlive=300
```

**Explicação dos parâmetros:**
- `CallTimeout`: Timeout em segundos para chamadas ao banco de dados (padrão: 30s)
- `CustomQueriesPath`: Caminho para armazenar arquivos SQL personalizados
- `KeepAlive`: Intervalo em segundos para manter a conexão ativa (padrão: 300s)

### Verificar configuração

```bash
grep -v "^#" /etc/zabbix/zabbix_agent2.conf | grep "Plugins.Oracle"
```

**Saída esperada:**
```
Plugins.Oracle.CallTimeout=30
Plugins.Oracle.CustomQueriesPath=/home/zabbix/sql
Plugins.Oracle.KeepAlive=300
```

---

## 6. Criando uma Query SQL para Teste

### Criar diretório de queries

```bash
mkdir -p /home/zabbix/sql
sudo chown zabbix:zabbix /home/zabbix/sql
sudo chmod 755 /home/zabbix/sql
```

### Criar arquivo de query

```bash
sudo cat > /home/zabbix/sql/check_oracle_version.sql << 'EOF'
select VERSION from V$INSTANCE
EOF
```

### Ajustar permissões

```bash
sudo chown zabbix:zabbix /home/zabbix/sql/check_oracle_version.sql
sudo chmod 644 /home/zabbix/sql/check_oracle_version.sql
```

---

## 7. Testando o Item no Zabbix

### Parâmetros de teste

Utilize os seguintes parâmetros para testar a conexão:

| Parâmetro | Valor |
|-----------|-------|
| **Connection String** | `tcp://localhost:1521` |
| **Username** | `zabbix_mon` |
| **Password** | `Zabbix321` |
| **Database** | `WINT` |
| **Query** | `check_oracle_version` |

### Sintaxe do item

```
oracle.custom.query[tcp://localhost:1521,zabbix_mon,Zabbix321,WINT,check_oracle_version]
```

### Saída esperada

```
oracle.custom.query[tcp://localhost:1521,zabbix_mon,Zabbix321,WINT,check_oracle_version][s|[{"VERSION":"11.2.0.4.0"}]]
```

**Componentes da resposta:**
- `s|`: Indica sucesso (string return)
- `[{"VERSION":"11.2.0.4.0"}]`: Resultado em formato JSON contendo a versão do banco de dados

---

## 8. Verificação de Conectividade

### Testar conexão antes de usar no Zabbix

```bash
# Usar sqlplus
sqlplus zabbix_mon/Zabbix321@localhost:1521/WINT

# Ou via Zabbix Agent 2 (requer agent rodando e Oracle (12c)+)
zabbix_agent2 -t "oracle.ping[tcp://localhost:1521,zabbix_mon,Zabbix321,WINT]"
```

---

## 9. Troubleshooting

### Erro: "Connection refused"

**Verificar:**
1. Listener Oracle está rodando: `lsnrctl status`
2. Porta 1521 está acessível: `netstat -an | grep 1521`
3. Connection string está correta

### Erro: "ORA-12514: TNS:listener does not currently know of service"

**Verificar:**
- O nome do banco de dados (WINT) está registrado no listener
- Usar `tnsping WINT` para testar a resolução de nome
- Usar `listener status` para verificar se está registrado

### Zabbix Agent 2 não inicia

**Verificar logs:**
```bash
# Ver status do serviço
sudo systemctl status zabbix-agent2

# Ver logs detalhados
sudo journalctl -u zabbix-agent2 -f

# Testar configuração
sudo zabbix_agent2 -c /etc/zabbix/zabbix_agent2.conf -t "agent.version"
```

### Permissões insuficientes no diretório /home/zabbix/sql

**Solução:**
```bash
sudo chown -R zabbix:zabbix /home/zabbix/sql
sudo chmod 755 /home/zabbix/sql
```

---

## Referências

- [Oracle Instant Client Documentation](https://docs.oracle.com/en/database/oracle/oracle-database/index.html)
- [Zabbix Agent 2 Oracle Plugin](https://git.zabbix.com/projects/ZBX/repos/zabbix/browse/src/go/plugins/oracle/README.md)
- [Oracle SQLNet Configuration](https://docs.oracle.com/en/database/oracle/oracle-database/19/netrf/parameters-for-the-sqlnet.ora.html)

---

**Última atualização:** Abril 2026
**Versões testadas:** OL6, OL7, OL8, OL9 | Oracle Instant Client 18.3, 19.23 | Oracle Database 11g, 12c, 19c | Zabbix Agent 2 5.4, 7.0
