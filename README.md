# Deploy da API Java na Azure VM (AlmaLinux)

Este guia documenta o passo a passo para subir uma API Java 25 em uma VM AlmaLinux na Azure, incluindo os erros encontrados e como foram resolvidos.

---

## Pré-requisitos

- VM AlmaLinux criada na Azure com IP público
- Acesso SSH à VM
- Projeto Maven com Java 25

---

## 1. Verificar o Java instalado

```bash
java -version
javac -version
```

### ⚠️ Erro encontrado: JDK 17 conflitando com JRE 25

A VM tinha o **JRE 25** instalado (usado pelo `java`) mas o **JDK 17** sendo usado pelo `javac` e pelo Maven. Como o Maven usa o `javac` para compilar, ele tentava compilar com Java 17 mas o projeto exigia Java 25, causando o erro:

```
Fatal error compiling: error: release version 25 not supported
```

### ✅ Solução: instalar o JDK 25

```bash
# Verificar se o pacote está disponível
sudo dnf search java-25

# Instalar o JDK 25
sudo dnf install java-25-openjdk-devel
```

---

## 2. Configurar o JAVA_HOME

Após instalar o JDK 25, é necessário apontar o `JAVA_HOME` para ele, caso contrário o Maven ainda usará o Java 17.

```bash
# Verificar o path correto do JDK 25
ls /usr/lib/jvm/ | grep java-25

# Exportar o JAVA_HOME (ajuste o path se necessário)
export JAVA_HOME=/usr/lib/jvm/java-25-openjdk
export PATH=$JAVA_HOME/bin:$PATH
```

Confirmar que tudo está apontando para Java 25:

```bash
java -version    # deve mostrar 25
javac -version   # deve mostrar 25
mvn -version     # deve mostrar Java version: 25
```

### Tornar permanente

```bash
echo 'export JAVA_HOME=/usr/lib/jvm/java-25-openjdk' >> ~/.bashrc
echo 'export PATH=$JAVA_HOME/bin:$PATH' >> ~/.bashrc
source ~/.bashrc
```

---

## 3. Configurar o pom.xml

Garantir que o `pom.xml` do projeto está configurado para Java 25:

```xml
<properties>
    <maven.compiler.release>25</maven.compiler.release>
    <java.version>25</java.version>
</properties>

<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.13.0</version>
            <configuration>
                <release>25</release>
            </configuration>
        </plugin>
    </plugins>
</build>
```

> **Nota:** Se o projeto usar features em preview do Java 25, adicionar `--enable-preview` nas `compilerArgs`.

---

## 4. Compilar o projeto

```bash
mvn clean package
```

### ⚠️ Aviso esperado (não é erro)

O Maven 3.6.3 exibe warnings ao rodar com Java 25 por causa de incompatibilidade com uma biblioteca interna (`jansi`). Esses avisos **não impedem o build**:

```
WARNING: A restricted method in java.lang.System has been called
WARNING: java.lang.System::load has been called by org.fusesource.jansi.internal.JansiLoader
```

---

## 5. Liberar a porta na Azure

Antes de subir a API, é necessário liberar a porta no **Network Security Group (NSG)** da Azure.

1. Acesse o [Portal Azure](https://portal.azure.com)
2. Vá em **Virtual Machines → sua VM**
3. No menu lateral, clique em **Networking → Network settings**
4. Clique em **+ Create port rule → Inbound port rule**
5. Preencha:
   - **Destination port ranges:** `8080`
   - **Protocol:** TCP
   - **Action:** Allow
   - **Priority:** `330`
   - **Name:** `Allow-8080`
6. Clique em **Add**

### ⚠️ Observação: firewalld não instalado

Na VM AlmaLinux, o `firewalld` não estava instalado, então **não é necessário** liberar a porta no SO. A regra da Azure é suficiente:

```bash
sudo firewall-cmd --permanent --add-port=8080/tcp  # Este comando NÃO é necessário
```

---

## 6. Subir a API

```bash
java -jar target/seu-arquivo.jar
```

Se quiser suprimir os warnings de native access:

```bash
java --enable-native-access=ALL-UNNAMED -jar target/seu-arquivo.jar
```

---

## 7. Acessar o Swagger

Com a API no ar, acesse pelo navegador:

```
http://<IP_PUBLICO>:8080/swagger-ui/index.html
```

Exemplo com o IP da VM:

```
http://172.208.49.111:8080/swagger-ui/index.html
```

> Outras URLs possíveis dependendo da configuração do projeto:
> - `http://172.208.49.111:8080/swagger-ui.html`
> - `http://172.208.49.111:8080/v3/api-docs`

---

## Resumo dos Erros e Soluções

| Erro | Causa | Solução |
|------|-------|---------|
| `release version 25 not supported` | Maven usando JDK 17 em vez do 25 | Instalar `java-25-openjdk-devel` e configurar `JAVA_HOME` |
| `javac -version` retornando 17 | JDK 25 não instalado, apenas JRE 25 | `sudo dnf install java-25-openjdk-devel` |
| `mvn -version` mostrando Java 17 | `JAVA_HOME` apontando para Java 17 | `export JAVA_HOME=/usr/lib/jvm/java-25-openjdk` |
| `firewall-cmd: command not found` | `firewalld` não instalado na VM | Não necessário — liberar porta apenas no NSG da Azure |

---

Link do Vídeo -> https://youtu.be/pqYnnDQDRpE
