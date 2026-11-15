# README Tâche 3

# Documentation GA 
### Documentation et justification des choix de conception et des choix d’implémentation pour cette modification de workflow

L'objectif est d'intégrer une analyse de sécurité au workflow GitHub Actions afin d'empêcher l'introduction de nouvelles vulnérabilités dans le projet Tika.

Semgrep a été retenu pour effectuer cette analyse statique. Une comparaison avec une version de référence (baseline) permet de distinguer les nouvelles vulnérabilités de celles déjà présentes dans le projet, évitant ainsi de bloquer le build pour des problèmes préexistants.

L'analyse est exécutée avant le build afin qu'une nouvelle vulnérabilité détectée puisse interrompre le processus. Des rapports différentiels et complets sont également conservés pour identifier les nouvelles vulnérabilités et obtenir une vue d'ensemble des résultats.

Le job `semgrep` a été ajouté dans la section `jobs` du fichier `.github/workflows/main-jdk17-build.yml`. Il s'exécute sur `ubuntu-latest` et possède une limite de 30 minutes. 

**Configuration de Semgrep**

La version utilisée et les ensembles de règles sont définis dans les variables d'environnement du job :

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

Le paramètre `fetch-depth: 0` permet de récupérer l'historique Git complet du dépôt, nécessaire pour ecomparer avec le commit de référence.

```yaml
      - uses: actions/checkout@v6
        with:
          fetch-depth: 0
```

Le workflow détermine ensuite le commit de référence selon le type d'événement. Pour une pull request, il utilise `github.event.pull_request.base.sha`. Pour un push, il utilise `github.event.before`. Si le commit est absent, invalide ou correspond à une nouvelle branche, le commit parent est utilisé comme solution de repli.

**Analyse des nouvelles vulnérabilités**

Une fois la référence déterminée, Semgrep compare le code courant à celle-ci avec `--baseline-commit`. L'option `--error` permet de faire échouer la commande lorsqu'une nouvelle vulnérabilité est détectée.

```yaml
          semgrep scan $SEMGREP_RULES --metrics=off \
            --baseline-commit "$BASELINE_SHA" \
            --error \
            --json-output="$RUNNER_TEMP/semgrep-reports/semgrep-nouvelles.json" \
            --sarif-output="$RUNNER_TEMP/semgrep-reports/semgrep-nouvelles.sarif"
```

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

Les rapports sont ensuite téléversés comme artefacts GitHub Actions, également avec `if: always()`, afin de pouvoir les consulter après l'exécution du workflow.

**Dépendance avec le build**

Finalement, le job `build` possède la dépendance suivante :

```yaml
  build:
    needs: semgrep
```

Cette dépendance garantit que le build attend la réussite du job Semgrep avant de commencer.

### Documentation et justification de la validation des modifications 

# Documentation vulnérabilités	
[Documenter, justifier, expliquer chaque vulnérabilité insérée:]
### Vulnérabilité 1
<table>
  <tr>
    <td> Choix du fichier modifié </td>
    <td></td>
  </tr>
  <tr>
    <td> Type de vulnérabilité </td>
    <td></td>
  </tr>
  <tr>
    <td> Raison pour la vulnérabilité </td>
    <td></td>
  </tr>
  <tr>
    <td> Résultat obtenu avec Semgre </td>
    <td></td>
  </tr>
</table>

### Vulnérabilité 2
<table>
  <tr>
    <td> Choix du fichier modifié </td>
    <td></td>
  </tr>
  <tr>
    <td> Type de vulnérabilité </td>
    <td></td>
  </tr>
  <tr>
    <td> Raison pour la vulnérabilité </td>
    <td></td>
  </tr>
  <tr>
    <td> Résultat obtenu avec Semgre </td>
    <td></td>
  </tr>
</table>

### Vulnérabilité 3
<table>
  <tr>
    <td> Choix du fichier modifié </td>
    <td></td>
  </tr>
  <tr>
    <td> Type de vulnérabilité </td>
    <td></td>
  </tr>
  <tr>
    <td> Raison pour la vulnérabilité </td>
    <td></td>
  </tr>
  <tr>
    <td> Résultat obtenu avec Semgre </td>
    <td></td>
  </tr>
</table>

### Vulnérabilité 4
<table>
  <tr>
    <td> Choix du fichier modifié </td>
    <td></td>
  </tr>
  <tr>
    <td> Type de vulnérabilité </td>
    <td></td>
  </tr>
  <tr>
    <td> Raison pour la vulnérabilité </td>
    <td></td>
  </tr>
  <tr>
    <td> Résultat obtenu avec Semgre </td>
    <td></td>
  </tr>
</table>

### Vulnérabilité 5
<table>
  <tr>
    <td> Choix du fichier modifié </td>
    <td></td>
  </tr>
  <tr>
    <td> Type de vulnérabilité </td>
    <td></td>
  </tr>
  <tr>
    <td> Raison pour la vulnérabilité </td>
    <td></td>
  </tr>
  <tr>
    <td> Résultat obtenu avec Semgre </td>
    <td></td>
  </tr>
</table>

### Vulnérabilité 6
<table>
  <tr>
    <td> Choix du fichier modifié </td>
    <td></td>
  </tr>
  <tr>
    <td> Type de vulnérabilité </td>
    <td></td>
  </tr>
  <tr>
    <td> Raison pour la vulnérabilité </td>
    <td></td>
  </tr>
  <tr>
    <td> Résultat obtenu avec Semgre </td>
    <td></td>
  </tr>
</table>

### Vulnérabilité 7
<table>
  <tr>
    <td> Choix du fichier modifié </td>
    <td></td>
  </tr>
  <tr>
    <td> Type de vulnérabilité </td>
    <td></td>
  </tr>
  <tr>
    <td> Raison pour la vulnérabilité </td>
    <td></td>
  </tr>
  <tr>
    <td> Résultat obtenu avec Semgre </td>
    <td></td>
  </tr>
</table>

### Comparaison des rapports Semgrep obtenus avant les modifications et après l’introduction des vulnérabilités
[Comparer le rapport Semgrep obtenu avant les modifications avec le rapport obtenu après l’introduction des vulnérabilités, afin de montrer quelles vulnérabilités sont nouvellement introduites]

