# FleetCheck – Build Systems Lab

This project is intentionally incomplete. Follow the worksheet in the order given.

Expected final application output:

```
FleetCheck 1.0
Vehicles loaded: 4
Vehicles requiring service: 2
Average mileage: 37000 km
```

Do not copy the solution POM. The objective is to observe how each build change alters the result.

---

## Relatório – evidências e respostas (Maven)

Ambiente: JDK 21.0.9 (Microsoft OpenJDK), Apache Maven 3.9.11, Git 2.55.

### Passo 1 – Build antes de qualquer alteração
`mvn clean package` falha na compilação:

```
[ERROR] /.../src/main/java/pt/upt/fleetcheck/App.java:[4,38] package com.fasterxml.jackson.databind does not exist
```

**Evidência 1:** o erro vem do `import com.fasterxml.jackson.databind.ObjectMapper;` (linha 4 de `App.java`);
a linha 3 (`import com.fasterxml.jackson.core.type.TypeReference;`) dá o erro equivalente para
`com.fasterxml.jackson.core.type`. O `pom.xml` não declarava a dependência Jackson.

### Passo 2 – Dependência explícita
Depois de acrescentar `jackson-databind:2.22.2`, a compilação passa mas os testes falham
(foi criado `src/test/java/pt/upt/fleetcheck/FleetServiceTest.java`, porque o starter não trazia testes):

```
[ERROR]   FleetServiceTest.countsVehiclesNeedingService:40 expected: <2> but was: <1>
[ERROR]   FleetServiceTest.vehicleExactlyAtIntervalNeedsService:24 expected: <true> but was: <false>
[ERROR] Tests run: 6, Failures: 2, Errors: 0, Skipped: 0
```

O defeito era de fronteira em `FleetService.needsService`: `>` em vez de `>=`. O veículo V3 tem
exatamente 10 000 km desde a última revisão (= intervalo) e deve precisar de revisão.
Corrigido para `>=` → `Tests run: 6, Failures: 0` e `BUILD SUCCESS`.

**Pergunta – porque é uma falha melhor do que a do passo 1?** A do passo 1 era um problema de
configuração do build (dependência não declarada): o build nem chegava a compilar, por isso não dizia
nada sobre o comportamento do software. No passo 2 o build já compila e chega à fase `test`, onde
um teste automático deteta um defeito funcional real (regra de negócio errada no limite). O build
passou a servir de *quality gate*: impede que código com comportamento incorreto seja empacotado.

### Passo 3 – `mvn dependency:tree`

```
pt.upt.softwarequality:fleetcheck:jar:1.0.0
+- com.fasterxml.jackson.core:jackson-databind:jar:2.22.2:compile
|  +- com.fasterxml.jackson.core:jackson-annotations:jar:2.22:compile
|  \- com.fasterxml.jackson.core:jackson-core:jar:2.22.2:compile
\- org.junit.jupiter:junit-jupiter:jar:5.14.4:test (...)
```

`jackson-databind` é dependência direta; `jackson-core` e `jackson-annotations` são transitivas
(dependências do próprio `jackson-databind`).

### Passo 4 – JAR executável
Com o JAR por omissão:

```
> java -jar target/fleetcheck-1.0.0.jar
no main manifest attribute, in target/fleetcheck-1.0.0.jar
> java -cp target/fleetcheck-1.0.0.jar pt.upt.fleetcheck.App
Caused by: java.lang.NoClassDefFoundError: com/fasterxml/jackson/core/type/TypeReference
```

Com o `maven-shade-plugin`:

```
> java -jar target/fleetcheck-1.0.0-all.jar
FleetCheck 1.0
Vehicles loaded: 4
Vehicles requiring service: 2
Average mileage: 37000 km
```

**Evidência 4:** o JAR por omissão (~6 KB) só tem as classes do projeto e um `MANIFEST.MF` sem
`Main-Class`. O Shade plugin cria um segundo artefacto, `fleetcheck-1.0.0-all.jar` (~2,3 MB,
classificador `all`): um *uber/fat JAR* que junta as classes do projeto com as de todas as dependências
de runtime (jackson-databind, jackson-core, jackson-annotations), e o `ManifestResourceTransformer` escreve
`Main-Class: pt.upt.fleetcheck.App` no manifest. O resultado é auto-suficiente e corre com `java -jar`.

### Passo 5 – Maven Wrapper
`mvn wrapper:wrapper` gerou `mvnw`, `mvnw.cmd` e `.mvn/wrapper/maven-wrapper.properties`
(fixado em `apache-maven-3.9.11`). Foi acrescentado `project.build.outputTimestamp=2026-09-27T00:00:00Z`.
`mvnw.cmd clean verify` → `BUILD SUCCESS`.

**Pergunta – que pressuposto escondido o wrapper removeu?** O de que cada máquina (de cada
programador ou do servidor de CI) já tem o Maven instalado, no `PATH`, e na versão certa. O wrapper
descarrega e usa sempre a versão fixada no projeto, por isso o build é igual em qualquer máquina.
(O `outputTimestamp` remove outro: a data/hora do build deixa de ficar gravada no JAR, o que ajuda a
builds reprodutíveis.)

### Passo 6 – GitHub Actions
Workflow em `.github/workflows/build.yml` (executa `./mvnw -B clean verify` e publica o artefacto
`fleetcheck-build` com o fat JAR, o relatório JaCoCo e o `bom.json`). Foi acrescentado o
`jacoco-maven-plugin` 0.8.15 (`prepare-agent` + `report` em `verify`) porque o workflow faz upload de
`target/site/jacoco/**`. O `mvnw` foi marcado como executável (`git update-index --chmod=+x mvnw`).

Execução bem-sucedida (verde, artefacto `fleetcheck-build`): https://github.com/RodriMonteiroM/fleetcheck-maven/actions/runs/37233973351
Repositório: https://github.com/RodriMonteiroM/fleetcheck-maven

### Passo 7 – SBOM (CycloneDX)
`mvnw.cmd clean verify` gera `target/bom.json`, que contém `jackson-databind`, `jackson-core` e
`jackson-annotations`.

**Evidência 7:** um SBOM descreve *tudo* o que vai dentro do software entregue, não apenas o que foi
escrito no `pom.xml`. O CycloneDX lê o grafo de dependências resolvido pelo Maven (o mesmo do
`dependency:tree`), que inclui as dependências transitivas trazidas pelo `jackson-databind`. Essas
bibliotecas também são executadas (e vão dentro do fat JAR), logo também podem ter vulnerabilidades ou
licenças relevantes e têm de constar do inventário.
