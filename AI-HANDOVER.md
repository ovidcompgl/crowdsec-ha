# AI Handover Checklist

Scop: acest document standardizeaza transferul proiectului catre AI-ul companiei, astfel incat sa poata intelege atat codul curent, cat si istoricul deciziilor.

## 1) Istoric obligatoriu de transferat
- Git history complet (fara export zip simplu)
- Toate branch-urile
- Toate tag-urile
- Mesajele de commit (fara squash excesiv)
- PR-uri, issue-uri, discutii tehnice (unde platforma permite export)

## 2) Context minim care trebuie sa existe in repo
- README actualizat (setup, rulare, dependinte)
- ADR/decizii arhitecturale (de ce a fost aleasa o solutie)
- Runbook operational (deploy, rollback, troubleshooting)
- Cronologie schimbari majore (milestone -> commit/tag)
- Mapare business -> tehnic (cerinta -> implementare)

## 3) Comenzi recomandate pentru transfer corect
```bash
# in repo sursa
git fetch --all --tags
git remote add company <URL_REPO_COMPANIE>
git push company --all
git push company --tags
```

Alternativa mirror (copie fidela):
```bash
git clone --mirror <URL_Sursa>
cd <repo>.git
git remote set-url origin <URL_REPO_COMPANIE>
git push --mirror
```

## 4) Verificari dupa import
- Exista aceleasi branch-uri ca in sursa
- Exista aceleasi tag-uri ca in sursa
- Commit count pe branch-urile principale este compatibil
- Ultimul commit pe main/master este acelasi SHA

## 5) Securitate inainte de predare
- Eliminare secrete hardcodate din fisiere
- Rotire credentiale expuse anterior
- Folosire secret manager / vault pentru parole
- Revizuire `.env`, YAML-uri de deploy si pipeline-uri CI

## 6) Ce indexeaza AI-ul companiei
AI-ul trebuie sa indexeze:
- codul sursa
- istoricul git
- documentatia operationala
- issue-uri/PR-uri (daca este disponibil conectorul)

Daca indexeaza doar fisierele curente, va pierde motivatia istorica a schimbarilor.

## 7) Regula de lucru recomandata
Pentru orice schimbare majora, adauga in acelasi PR:
- ce s-a schimbat
- de ce s-a schimbat
- impact/risc
- plan de rollback

Acest format creste calitatea contextului pentru AI si reduce timpul de onboarding.
