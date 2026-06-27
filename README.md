# [Tansoftware](https://www.tansoftware.com) - Fonctionnement d'un compilateur [![fr](https://raw.githubusercontent.com/gosquared/flags/master/flags/flags/shiny/24/France.png)](README.md)

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE) [![Lang](https://img.shields.io/badge/Lang-Français-005EB8.svg)](#) [![Topic](https://img.shields.io/badge/Topic-Compilers-brightgreen.svg)](#)

## Table des matières

1. [Fondations et vue d'ensemble](chapitres/01-fondations-et-vue-densemble.md)
   - [Introduction](chapitres/01-fondations-et-vue-densemble.md#introduction)
   - [Glossaire express](chapitres/01-fondations-et-vue-densemble.md#glossaire-express)
   - [Vue d'ensemble du pipeline](chapitres/01-fondations-et-vue-densemble.md#vue-densemble-du-pipeline)
2. [Le front-end : de la source a l'arbre](chapitres/02-le-front-end-de-la-source-a-larbre.md)
   - [Analyse lexicale](chapitres/02-le-front-end-de-la-source-a-larbre.md#analyse-lexicale)
   - [Analyse syntaxique](chapitres/02-le-front-end-de-la-source-a-larbre.md#analyse-syntaxique)
   - [Arbre syntaxique abstrait (AST)](chapitres/02-le-front-end-de-la-source-a-larbre.md#arbre-syntaxique-abstrait-ast)
   - [Analyse sémantique](chapitres/02-le-front-end-de-la-source-a-larbre.md#analyse-sémantique)
3. [Representation intermediaire et optimisation](chapitres/03-representation-intermediaire-et-optimisation.md)
   - [Génération de code intermédiaire](chapitres/03-representation-intermediaire-et-optimisation.md#génération-de-code-intermédiaire)
   - [Forme SSA](chapitres/03-representation-intermediaire-et-optimisation.md#forme-ssa)
   - [Optimisation du code](chapitres/03-representation-intermediaire-et-optimisation.md#optimisation-du-code)
4. [Le back-end : du code machine a l'executable](chapitres/04-le-back-end-du-code-machine-a-lexecutable.md)
   - [Allocation de registres](chapitres/04-le-back-end-du-code-machine-a-lexecutable.md#allocation-de-registres)
   - [Génération du code machine](chapitres/04-le-back-end-du-code-machine-a-lexecutable.md#génération-du-code-machine)
   - [Éditeur de liens et chargement](chapitres/04-le-back-end-du-code-machine-a-lexecutable.md#éditeur-de-liens-et-chargement)
5. [Compilateurs modernes et execution](chapitres/05-compilateurs-modernes-et-execution.md)
   - [Les compilateurs modernes](chapitres/05-compilateurs-modernes-et-execution.md#les-compilateurs-modernes)
   - [JIT, AOT et compilation adaptative](chapitres/05-compilateurs-modernes-et-execution.md#jit-aot-et-compilation-adaptative)
6. [Mise en pratique](chapitres/06-mise-en-pratique.md)
   - [Étude de cas : C → AST → 3-adresses → assembleur](chapitres/06-mise-en-pratique.md#étude-de-cas-c-ast-3-adresses-assembleur)
   - [Gestion de la mémoire et runtime](chapitres/06-mise-en-pratique.md#gestion-de-la-mémoire-et-runtime)
   - [Diagnostics et messages d'erreur](chapitres/06-mise-en-pratique.md#diagnostics-et-messages-derreur)
7. [Sujets avances](chapitres/07-sujets-avances.md)
   - [Bootstrapping et confiance](chapitres/07-sujets-avances.md#bootstrapping-et-confiance)
   - [WebAssembly et générateurs de code alternatifs](chapitres/07-sujets-avances.md#webassembly-et-générateurs-de-code-alternatifs)

---

## Pour aller plus loin

Lectures recommandées, par ordre approximatif de difficulté :

- *Crafting Interpreters*, Robert Nystrom ([disponible en ligne](https://craftinginterpreters.com/)). Construit deux interpréteurs complets pour le langage Lox. Excellente porte d'entrée.
- *Modern Compiler Implementation in Java/ML/C* (« Tiger Book »), Andrew W. Appel. Construction guidée d'un compilateur Tiger, du lexer à la génération de code.
- *Engineering a Compiler*, Keith D. Cooper, Linda Torczon. Très bon équilibre théorie / pratique, accent sur l'optimisation et SSA.
- *Compilers: Principles, Techniques, and Tools* (« Dragon Book »), Aho, Lam, Sethi, Ullman. La référence pour la théorie des langages, le lexing et le parsing.
- *Advanced Compiler Design and Implementation*, Steven Muchnick. Le complément back-end / optimisations du Dragon.
- *Static Single Assignment Book*, Rastello & Bouchez Tichadou (libre, [pfalcon.github.io/ssabook](http://ssabook.gforge.inria.fr/latest/book.pdf)). Tout sur SSA et ses transformations.
- *Linkers and Loaders*, John R. Levine. Sur la liaison statique et dynamique, ELF/PE/Mach-O.
- *Types and Programming Languages*, Benjamin Pierce. Pour le typage, l'inférence, la sémantique.

Documentations et cours :

- [LLVM Project](https://llvm.org/) : l'infrastructure de compilation moderne (LLVM IR, *passes*, ORC JIT).
- [GCC Internals manual](https://gcc.gnu.org/onlinedocs/gccint/) : architecture interne de GCC, GIMPLE, RTL.
- [Stanford CS143 Compilers](https://web.stanford.edu/class/cs143/) : cours en ligne complet, projet COOL.
- [Cornell CS 6120 Advanced Compilers](https://www.cs.cornell.edu/courses/cs6120/) : cours moderne centré sur SSA et l'optimisation.
- [V8 blog](https://v8.dev/blog) et [HotSpot Wiki](https://wiki.openjdk.org/display/HotSpot) : articles d'ingénieurs sur les JIT en production.

## Licence

Distribué sous licence [MIT](LICENSE).

## Auteur

**Tansoftware - Tanguy Chénier** · [LinkedIn](https://www.linkedin.com/in/tanguy-chenier) · [Tan-Software](https://github.com/Tan-Software) · [Compte personnel (derniers outils)](https://github.com/tanguychenier) · [tansoftware.com](https://www.tansoftware.com)
