# README Tâche 3

Le dépôt avec le workflow modifié : https://github.com/warrupp/IFT3913_T3

Les fichiers qu'on a modifiés pour la GitHub Action sont [`.github/workflows/main-jdk17-build.yml`](https://github.com/warrupp/IFT3913_T3/blob/main/.github/workflows/main-jdk17-build.yml) et [`.github/actions/rickroll/action.yml`](https://github.com/warrupp/IFT3913_T3/blob/main/.github/actions/rickroll/action.yml) (pour le rickroll).

# Documentation GA

### Documentation et justification des choix de conception et des choix d'implémentation pour cette modification de workflow

L'objectif est d'intégrer une analyse de sécurité au workflow GitHub Actions afin d'empêcher l'introduction de nouvelles vulnérabilités dans le projet Tika.

Comme demandé dans l'énoncé, l'analyse statique est faite avec Semgrep. Avant de modifier quoi que ce soit, on a lancé Semgrep sur le code original de Tika et il trouvait déjà 19 alertes. Si on faisait juste échouer le build dès que Semgrep trouve quelque chose, le build aurait échoué à tous les commits. C'est pour ça qu'on compare avec une version de référence (baseline) : comme ça seules les nouvelles vulnérabilités font échouer le build, et pas celles qui étaient déjà là.

L'analyse est exécutée avant le build afin qu'une nouvelle vulnérabilité détectée puisse interrompre le processus. Des rapports différentiels et complets sont également conservés pour identifier les nouvelles vulnérabilités et obtenir une vue d'ensemble des résultats.

Le job `semgrep` a été ajouté dans la section `jobs` du fichier `.github/workflows/main-jdk17-build.yml`. Il s'exécute sur `ubuntu-latest` et possède une limite de 30 minutes.

**Configuration de Semgrep**

La version utilisée et les ensembles de règles sont définis dans les variables d'environnement du job. On a fixé la version pour avoir toujours les mêmes résultats, même si Semgrep sort une nouvelle version entre deux runs.

```yaml
    env:
      SEMGREP_VERSION: '1.185.0'
      SEMGREP_RULES: '--config p/java --config p/security-audit --config p/secrets'
```

Semgrep est ensuite installé à l'aide de `pipx` :

```yaml
      - name: Install Semgrep
        run: pipx install "semgrep==${SEMGREP_VERSION}"
```

**Récupération de la version de référence**

Le paramètre `fetch-depth: 0` permet de récupérer l'historique Git complet du dépôt, nécessaire pour comparer avec le commit de référence.

```yaml
      - uses: actions/checkout@v6
        with:
          fetch-depth: 0
```

Le workflow détermine ensuite le commit de référence selon le type d'événement. Pour une pull request, il utilise `github.event.pull_request.base.sha`. Pour un push, il utilise `github.event.before`. Si le commit est absent, invalide ou correspond à une nouvelle branche (dans ce cas GitHub donne `000...0`), le commit parent est utilisé comme solution de repli. On passe ces valeurs au script par des variables d'environnement au lieu de les mettre directement dans la commande, c'est ce qui est recommandé par GitHub pour éviter les problèmes d'injection dans les scripts.

**Analyse des nouvelles vulnérabilités**

Une fois la référence déterminée, Semgrep compare le code courant à celle-ci avec `--baseline-commit`. L'option `--error` permet de faire échouer la commande lorsqu'une nouvelle vulnérabilité est détectée.

```yaml
          semgrep scan $SEMGREP_RULES --metrics=off \
            --baseline-commit "$BASELINE_SHA" \
            --error \
            --json-output="$RUNNER_TEMP/semgrep-reports/semgrep-nouvelles.json" \
            --sarif-output="$RUNNER_TEMP/semgrep-reports/semgrep-nouvelles.sarif"
```

On écrit les rapports dans `$RUNNER_TEMP`, donc en dehors du dépôt. Au début ils étaient dans le dossier du projet, mais dans ce cas le deuxième scan aurait analysé les fichiers JSON du premier.

**Génération et conservation des rapports**

En complément de l'analyse différentielle, une analyse complète génère des rapports sur les vulnérabilités présentes dans le projet. Elle utilise `if: always()` afin de s'exécuter même si l'analyse différentielle échoue. Contrairement à cette dernière, elle sert à produire un rapport et non à bloquer le build.

```yaml
      - name: Semgrep full report
        if: always()
        run: |
          semgrep scan $SEMGREP_RULES --metrics=off \
            --json-output="$RUNNER_TEMP/semgrep-reports/semgrep-complet.json" \
            --sarif-output="$RUNNER_TEMP/semgrep-reports/semgrep-complet.sarif" \
            --quiet > /dev/null
```

Les rapports sont ensuite téléversés comme artefacts GitHub Actions (`semgrep-reports`), également avec `if: always()`, afin de pouvoir les consulter après l'exécution du workflow.

**Dépendance avec le build**

Finalement, le job `build` possède la dépendance suivante :

```yaml
  build:
    needs: semgrep
```

Cette dépendance garantit que le build attend la réussite du job Semgrep avant de commencer. Donc si Semgrep échoue, le build et les tests e2e ne sont même pas lancés.

**Exclusion du test d'intégration S3**

On a aussi dû faire une autre modification. La première fois qu'on a lancé le workflow, Semgrep passait mais le build échouait. En regardant les logs on a vu que c'était le test `tika-pipes-s3-integration-tests` qui plantait parce qu'il n'arrivait pas à télécharger l'image Docker `quay.io/minio/minio` (erreur `401 UNAUTHORIZED`). On a relancé le run et ça a fait la même chose, l'image n'est plus disponible publiquement. Comme ça n'a rien à voir avec nos changements et que ça nous empêchait de voir si le reste marchait, on a enlevé ce module de la commande Maven avec `-pl` :

```yaml
        run: mvn clean apache-rat:check test install javadoc:aggregate -Pci -B -pl '!tika-integration-tests/tika-pipes-s3-integration-tests' ...
```

Après ça, le build passait au complet.

### Documentation et justification de la validation des modifications

Pour valider, on a poussé des commits sur `main` et on a regardé les runs sur GitHub.

Pour le cas où il n'y a pas de nouvelle vulnérabilité, on a utilisé le commit qui enlève le test S3, qui touche seulement au workflow. Le job `semgrep` a analysé juste le fichier modifié, il a trouvé 0 alerte (`Findings: 0 (0 blocking)`), et ensuite le build et les tests e2e ont passé : https://github.com/warrupp/IFT3913_T3/actions/runs/43756090639

Pour le cas inverse, le commit qui ajoute la vulnérabilité 1 (injection de script) fait bien échouer le job `semgrep`, et le build et les tests e2e sont sautés : https://github.com/warrupp/IFT3913_T3/actions/runs/43777379440. Voici la partie importante du log :

```
Baseline commit: 5228dc33b22c177e6bf1139aded19613e0404107
  Current version has 1 finding.
    tika-core/src/main/java/org/apache/tika/metadata/filter/GeoPointMetadataFilter.java
    ❯❱ java.lang.security.audit.script-engine-injection.script-engine-injection
 • Findings: 1 (1 blocking)
##[error]Process completed with exit code 1.
```

On voulait aussi vérifier que les alertes qui existaient déjà ne sont pas comptées. Dans le commit où on ajoute les 6 autres vulnérabilités, Semgrep trouve 10 alertes dans les fichiers modifiés, mais il en garde seulement 8 comme nouvelles. Les 2 qui restent (`insecure-hostname-verifier` et `use-of-sha1` dans `HttpClientFactory.java`) étaient déjà dans le code de Tika avant nous, donc la baseline fonctionne bien : https://github.com/warrupp/IFT3913_T3/actions/runs/43778699244

On a aussi eu un cas qui n'a pas marché. Au début, notre première vulnérabilité était une injection de commande dans `ExternalEmbedder.java` (`Runtime.getRuntime().exec` avec `sh -c`). On s'attendait à ce que le job échoue mais il a passé : https://github.com/warrupp/IFT3913_T3/actions/runs/43775845466. Dans le log on voyait que Semgrep avait bien analysé le fichier, donc ce n'était pas un problème du workflow, c'est la règle qui ne reconnaissait pas notre code. Pour éviter de refaire la même erreur, on a écrit un petit fichier Java de test avec plusieurs types de failles et on l'a passé dans Semgrep avec les mêmes règles que le CI pour voir lesquelles sont vraiment détectées. Ensuite on a remplacé l'injection de commande par une injection de script.

# Rickroll

Pour le rickroll on a fait notre propre action réutilisable dans `.github/actions/rickroll/action.yml` (une action `composite`). Quand elle se lance, elle affiche les paroles de « Never Gonna Give You Up » dans les logs, elle met une annotation rouge « Tests echoues » sur la page du run, et elle ajoute dans le résumé du job un lien « Voir le rapport detaille des tests echoues » qui envoie en fait sur https://watchbutdonotlearn.github.io/. Le lien et son texte sont des paramètres de l'action (`url` et `link-text`), comme ça on pourrait la réutiliser dans un autre workflow.

Dans le workflow, on l'appelle juste après l'étape Maven, et seulement si c'est cette étape qui a échoué :

```yaml
      - name: Build with Maven
        id: maven
        run: mvn ...

      - name: Rickroll
        if: failure() && steps.maven.outcome == 'failure'
        uses: ./.github/actions/rickroll
```

On a mis la condition sur `steps.maven.outcome` pour que le rickroll arrive seulement quand ce sont les tests qui échouent, et pas par exemple si l'installation de Java plante.

Pour tester, on a ajouté un test qui échoue exprès dans `tika-core` (`RickrollTest.java`, qui compare « never gonna give you up » avec « never gonna let you down »), puis on l'a enlevé une fois qu'on avait vu le résultat. Quand les tests passent il n'y a pas de rickroll (https://github.com/warrupp/IFT3913_T3/actions/runs/43760818869) et quand le test échoue le rickroll se déclenche (https://github.com/warrupp/IFT3913_T3/actions/runs/43765723410).

# Documentation vulnérabilités

Pour chaque vulnérabilité, on a essayé de choisir un fichier où la faille a du sens par rapport à ce que fait la classe. On a aussi évité les fichiers où Semgrep trouvait déjà la même alerte avant, pour que la comparaison avant/après soit claire. Dans le code, chaque modification est marquée avec un commentaire `TACHE 3 - VULNERABILITE X`. On a fait attention à ce que le code compile toujours et que les tests de Tika passent encore.

### Vulnérabilité 1 : injection de script (CWE-94)

On a modifié `tika-core/src/main/java/org/apache/tika/metadata/filter/GeoPointMetadataFilter.java`. Ce filtre prend la latitude et la longitude trouvées dans le document (par exemple les coordonnées GPS d'une photo) et les met ensemble. On a changé le code pour que ces valeurs passent par un moteur de script, avec l'idée de pouvoir écrire des coordonnées comme `45+30/60` :

```java
ScriptEngine engine = new ScriptEngineManager().getEngineByName("js");
...
return String.valueOf(engine.eval(valeur));
```

C'est vulnérable parce que `eval()` exécute la valeur comme du code, alors que la latitude et la longitude viennent du fichier analysé. Quelqu'un qui contrôle les métadonnées du fichier peut donc faire exécuter du code. Petite remarque : Java 17 n'a plus de moteur JavaScript intégré, donc dans notre build `engine` est `null` et la valeur est gardée telle quelle. Mais dès qu'un moteur de script est ajouté au projet, la faille est réelle.

Semgrep la détecte avec la règle `java.lang.security.audit.script-engine-injection` à la ligne 105.

### Vulnérabilité 2 : désérialisation non sûre (CWE-502)

On a modifié `tika-xmp/src/main/java/org/apache/tika/xmp/XMPMetadata.java`, la classe qui gère les métadonnées XMP. Elle touchait déjà à la sérialisation (elle bloque `readObject`), donc on a ajouté une méthode qui relit un cache de métadonnées sauvegardé dans un fichier :

```java
public static Object relireCache(InputStream in) throws IOException, ClassNotFoundException {
    ObjectInputStream ois = new ObjectInputStream(in);
    return ois.readObject();
}
```

Le problème c'est que `readObject()` recrée n'importe quelle classe qui se trouve dans le flux sans vérifier ce que c'est. Si quelqu'un modifie le fichier de cache, il peut faire créer des objets dangereux au moment où on le relit.

Semgrep : `java.lang.security.audit.object-deserialization` à la ligne 600.

### Vulnérabilité 3 : injection SQL (CWE-89)

On a modifié `tika-eval/tika-eval-app/src/main/java/org/apache/tika/eval/app/reports/Report.java`. tika-eval stocke ses résultats dans une base de données, et cette classe fait des requêtes SQL pour générer les rapports. On n'a pas pris `JDBCUtil.java` ou `H2Util.java` parce qu'ils avaient déjà une alerte SQL avant nous. On a ajouté une méthode qui compte les résultats pour un fichier :

```java
String sql = "SELECT COUNT(*) FROM profiles WHERE FILE_NAME = '" + fichier + "'";
return st.executeQuery(sql);
```

Le nom du fichier est collé directement dans la requête au lieu d'utiliser un `PreparedStatement` avec un `?`. Si la valeur contient une apostrophe, elle peut changer le sens de la requête.

Semgrep : `java.lang.security.audit.formatted-sql-string` à la ligne 75.

### Vulnérabilité 4 : XXE (CWE-611)

On a modifié `tika-parsers/tika-parsers-standard/tika-parsers-standard-modules/tika-parser-apple-module/src/main/java/org/apache/tika/parser/apple/PListParser.java`. Ce parser lit les fichiers plist d'Apple, qui peuvent être en XML, donc c'était logique d'y lire du XML :

```java
DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
DocumentBuilder db = dbf.newDocumentBuilder();
return db.parse(in);
```

On utilise le `DocumentBuilderFactory` de base sans désactiver les DOCTYPE. Un fichier XML piégé peut alors déclarer des entités externes, ce qui permet de lire des fichiers sur le serveur ou de lui faire faire des requêtes réseau pendant le parsing.

Semgrep : `java.lang.security.audit.xxe.documentbuilderfactory-disallow-doctype-decl-missing` à la ligne 101.

### Vulnérabilité 5 : TrustManager qui accepte tous les certificats (CWE-295)

On a modifié `tika-pipes/tika-httpclient-commons/src/main/java/org/apache/tika/client/HttpClientFactory.java`, la classe qui crée les clients HTTP(S) de tika-pipes et qui s'occupe des certificats. On a ajouté un TrustManager qui ne vérifie rien :

```java
new X509TrustManager() {
    public void checkClientTrusted(X509Certificate[] chain, String authType) {
    }
    public void checkServerTrusted(X509Certificate[] chain, String authType) {
    }
    public X509Certificate[] getAcceptedIssuers() {
        return null;
    }
}
```

Comme les méthodes de vérification sont vides, n'importe quel certificat est accepté. Quelqu'un qui se met entre le client et le serveur (man-in-the-middle) peut donc lire ou modifier les échanges.

Semgrep : `java.lang.security.audit.crypto.ssl.insecure-trust-manager` aux lignes 282, 285 et 288. On a 3 alertes pour une seule faille parce que la règle signale chaque méthode à part.

### Vulnérabilité 6 : hachage faible MD5 (CWE-328)

On a modifié `tika-core/src/main/java/org/apache/tika/digest/InputStreamDigester.java`, qui calcule les condensés (digests) des fichiers. On a ajouté une méthode « rapide » pour comparer deux contenus :

```java
return MessageDigest.getInstance("MD5").digest(contenu);
```

MD5 n'est plus considéré comme sûr parce qu'on sait créer deux contenus différents avec le même MD5 (collisions). Donc ce condensé ne garantit pas que deux fichiers sont vraiment les mêmes.

Semgrep : `java.lang.security.audit.crypto.use-of-md5` à la ligne 84.

### Vulnérabilité 7 : chiffrement faible DES (CWE-327)

On a modifié `tika-core/src/main/java/org/apache/tika/parser/CryptoParser.java`, le parser qui déchiffre des documents avec `Cipher`. On a ajouté un « ancien mode » pour la compatibilité :

```java
return Cipher.getInstance("DES/ECB/PKCS5Padding");
```

DES utilise une clé de 56 bits, ce qui se casse par force brute aujourd'hui. En plus, avec le mode ECB, deux blocs identiques donnent le même résultat chiffré, donc on peut voir des motifs dans les données.

Semgrep : `java.lang.security.audit.crypto.des-is-deprecated` à la ligne 77.

### Comparaison des rapports Semgrep obtenus avant les modifications et après l'introduction des vulnérabilités

Pour le rapport avant, on a lancé Semgrep sur le code original (commit `a2d75c2c1`) avec la même version et les mêmes règles que dans le CI. Pour le rapport après, on a pris le rapport complet `semgrep-complet.json` dans l'artefact du [dernier run](https://github.com/warrupp/IFT3913_T3/actions/runs/43778699244).

Le rapport avant contient 19 alertes et le rapport après en contient 28, donc 9 de plus. On a comparé les deux avec un petit script Python, par règle et par fichier. On n'a pas comparé les numéros de ligne parce qu'ils changent quand on ajoute du code dans un fichier : par exemple l'alerte `use-of-sha1` de `HttpClientFactory.java` était à la ligne 466 avant et à la ligne 490 après.

Les alertes qui apparaissent seulement dans le rapport après sont :
- `script-engine-injection` dans `GeoPointMetadataFilter.java` (ligne 105)
- `object-deserialization` dans `XMPMetadata.java` (ligne 600)
- `formatted-sql-string` dans `Report.java` (ligne 75)
- `documentbuilderfactory-disallow-doctype-decl-missing` dans `PListParser.java` (ligne 101)
- `insecure-trust-manager` dans `HttpClientFactory.java` (lignes 282, 285 et 288)
- `use-of-md5` dans `InputStreamDigester.java` (ligne 84)
- `des-is-deprecated` dans `CryptoParser.java` (ligne 77)

Ça fait bien 9 alertes pour nos 7 vulnérabilités (le TrustManager compte pour 3). Les 19 alertes du début sont toujours dans le rapport après et aucune n'a disparu, donc les nouvelles alertes sont exactement celles qu'on a ajoutées.
