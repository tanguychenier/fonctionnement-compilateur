[← Compilateurs modernes et execution](05-compilateurs-modernes-et-execution.md) · [↑ Sommaire](../README.md#table-des-matières) · [Sujets avances →](07-sujets-avances.md)

# 6. Mise en pratique

## Étude de cas : C → AST → 3-adresses → assembleur

Suivons une fonction C minimale à travers tout le pipeline.

### Source C

```c
int somme_carres(int n) {
    int s = 0;
    for (int i = 1; i <= n; i++) {
        s = s + i * i;
    }
    return s;
}
```

### AST (vue simplifiée)

```text
FunDecl somme_carres : (int n) -> int
└── Block
    ├── VarDecl s = Num(0)
    ├── For
    │   ├── init : VarDecl i = Num(1)
    │   ├── cond : LessEq(Var(i), Var(n))
    │   ├── step : Assign(i, Plus(Var(i), Num(1)))
    │   └── body : Assign(s, Plus(Var(s), Mul(Var(i), Var(i))))
    └── Return Var(s)
```

### Code à 3 adresses (TAC)

```text
somme_carres(n):
        s   = 0
        i   = 1
.L1:    if i > n goto .L2
        t1  = i * i
        s   = s + t1
        i   = i + 1
        goto .L1
.L2:    return s
```

### Forme SSA

```text
somme_carres(n):
        s0 = 0
        i0 = 1
.L1:    s1 = φ(s0, s2)
        i1 = φ(i0, i2)
        if i1 > n goto .L2
        t1 = i1 * i1
        s2 = s1 + t1
        i2 = i1 + 1
        goto .L1
.L2:    return s1
```

### Assembleur x86-64 (System V AMD64, sans optimisation puis avec)

Sans optimisation, `gcc -O0` produit un code mécanique fidèle au TAC, avec accès mémoire à chaque variable. Avec `-O2`, `s` et `i` vivent en registres et la boucle se réduit à :

```asm
somme_carres:
        test    edi, edi          ; n <= 0 ?
        jle     .Lzero
        mov     eax, 0            ; s = 0
        mov     ecx, 1            ; i = 1
.Lloop: mov     edx, ecx
        imul    edx, ecx          ; t1 = i * i
        add     eax, edx          ; s += t1
        add     ecx, 1            ; i++
        cmp     ecx, edi
        jle     .Lloop
        ret
.Lzero: xor     eax, eax
        ret
```

On voit le résultat de plusieurs passes : *constant folding* sur la borne, allocation de registres (`s`→`eax`, `i`→`ecx`), peephole (`mov eax,0` initial est conservé ici car `eax` est ensuite incrémenté ; `xor eax,eax` apparaît dans la branche zéro), choix d'instruction (`imul` plutôt qu'une séquence shift/add), respect de l'ABI (`edi` reçoit le premier argument, `eax` rend la valeur).

[Retour en haut de page](#table-des-matières)

## Gestion de la mémoire et runtime

> **Que veut dire « mémoire », « allocation », « runtime » et « modèle mémoire » ?** Pour fonctionner, un programme range ses données dans la mémoire de l'ordinateur. **Allouer** de la mémoire, c'est réserver une zone pour y mettre une donnée ; la **désallouer** (ou libérer), c'est rendre cette zone quand on n'en a plus besoin, afin qu'elle resserve. Le **runtime** est le petit programme de soutien embarqué avec votre code, qui assure ces tâches en coulisse pendant l'exécution. Le **modèle mémoire** décrit qui, du programmeur, du compilateur ou du runtime, a la charge d'allouer et de libérer. Image : la mémoire est un parking ; allouer = se garer, libérer = repartir, et le modèle dit qui surveille les places.

> **Que veut dire « write barrier » et « drop » ?** Une *write barrier* (« barrière d'écriture ») est un petit code ajouté automatiquement à chaque modification d'un objet pour prévenir le ramasse-miettes que quelque chose a changé, afin qu'il garde sa carte de la mémoire à jour. Un bloc `drop` est le code que des langages comme Rust insèrent à la sortie d'une portée pour libérer proprement ce qui ne sert plus, sans intervention du programmeur.

> **Que veut dire « modèle mémoire » selon les langages ?** Chaque langage choisit qui libère la mémoire et quand. Le choix conditionne le code émis par le back-end : barrières d'écriture pour le ramasse-miettes, blocs `drop` insérés à la sortie de portée, comptage de références implicite.

### Trois grandes familles

| Modèle | Qui libère ? | Surcoût d'exécution | Exemples |
|--------|--------------|---------------------|----------|
| **Manuel** | le programmeur (`malloc`/`free`, `new`/`delete`) | nul (mais bugs : *use-after-free*, *double-free*, fuites) | C, C++ historique, Zig |
| **RAII / *ownership*** | le compilateur, automatiquement à la sortie de portée | nul ou quasi | C++ moderne (RAII), Rust (*ownership*+*borrow*) |
| **ARC** (*Automatic Reference Counting*) | runtime, à la dernière référence | incrémenter/décrémenter à chaque copie ; cycles fuient | Swift, Objective-C ARC, Python (CPython) |
| **GC tracing** | runtime, périodique | pauses GC, *write barriers*, surcoût mémoire | Java, Go, C#, Haskell, OCaml, JavaScript |

> **Que veulent dire ces familles (manuel, RAII / ownership, ARC, GC) ?** En gestion **manuelle**, le programmeur réserve et rend la mémoire à la main : maximum de contrôle, maximum d'erreurs possibles (oublier de libérer, ou utiliser une zone déjà rendue). **RAII / ownership** (« acquisition de ressource = initialisation » et « propriété ») confie au compilateur la libération automatique dès qu'une donnée sort de sa portée, comme une lumière à détecteur qui s'éteint quand vous quittez la pièce. **ARC** (*Automatic Reference Counting*, « comptage automatique de références ») compte combien de personnes utilisent une donnée et la libère quand le compteur tombe à zéro, comme une salle qu'on ferme quand le dernier occupant sort. Le **GC** (*Garbage Collector*, « ramasse-miettes ») est un programme qui passe régulièrement repérer et jeter ce dont plus personne ne se sert, comme un service de nettoyage qui fait sa ronde.

### Algorithmes de garbage collection

Le compilateur d'un langage à GC émet du code coopératif avec le collecteur (carte mémoire, *safepoints*, *write barriers*). Les principales familles d'algorithmes :

> **Que veut dire « atteignable », « racines » et « fragmentation » ?** Les **racines** sont les points de départ connus (les variables en cours d'usage). Une donnée est **atteignable** si l'on peut y arriver en suivant les références depuis ces racines, comme tout ce qu'on peut joindre en suivant les liens d'un carnet d'adresses. Ce qui n'est plus atteignable est du déchet : personne ne pourra plus jamais y accéder, on peut le jeter. La **fragmentation** est l'état où la mémoire libre est éparpillée en petits trous entre des données encore utilisées, si bien qu'on ne trouve plus de grand espace continu, comme un parking plein de places isolées où un autocar ne peut pas se garer.

- **Mark-and-sweep** : marque tout ce qui est atteignable depuis les racines, puis libère le reste. Simple mais fragmente la mémoire.
- **Mark-compact** : après marquage, compacte les vivants en bord de tas. Élimine la fragmentation au prix d'une passe supplémentaire.
- **Copying / semi-spaces** (Cheney) : on partage le tas en deux moitiés, on copie les vivants d'une moitié à l'autre, on échange les rôles. Allocation triviale (bump pointer), mais on n'utilise que la moitié du tas.
- **Generational** : la plupart des objets meurent jeunes (*hypothèse générationnelle*). On alloue en *young generation*, on copie les survivants en *old generation*. La young est collectée fréquemment et vite.
- **Concurrent / incremental** : le GC tourne en parallèle du *mutator* (le programme), avec des *write barriers* pour suivre les modifications. Exemples : ZGC (Java), Shenandoah, Go (concurrent mark + sweep), V8 Orinoco.
- **Region-based** (MLton, Cyclone) : statique, le compilateur prouve qu'une région entière est libérable d'un bloc.

### Compile time vs run time

La frontière intéresse directement le compilateur :

- **Allocation au tas** : généralement runtime (`malloc`, `mmap`, GC bump pointer) ; le compilateur insère l'appel.
- ***Escape analysis*** : passe d'optimisation qui prouve qu'un objet ne s'échappe pas du cadre courant et peut donc être alloué en pile (HotSpot, Go, GraalVM).

> **Que veut dire « escape analysis » (analyse d'échappement) ?** C'est le compilateur qui se demande : « cet objet va-t-il survivre à la fonction qui le crée, ou bien rester confiné à l'intérieur ? ». S'il ne « s'échappe » pas (personne d'extérieur n'en gardera de référence), on peut le ranger dans la pile, libérée gratuitement à la sortie de la fonction, plutôt que dans le tas surveillé par le ramasse-miettes. C'est comme distinguer un brouillon jetable, qu'on déchire en quittant la pièce, d'un document à archiver.
- ***Stack maps*** : tables émises à la compilation décrivant, à chaque *safepoint*, l'emplacement des pointeurs vivants ; consommées par le GC à l'exécution.
- ***Drop glue*** : Rust émet à la compilation le code qui appellera les destructeurs à la sortie de portée. Pas de runtime requis.

[Retour en haut de page](#table-des-matières)

## Diagnostics et messages d'erreur

> **Que veut dire « diagnostic » ?** En médecine, un diagnostic explique ce qui ne va pas. Ici, c'est pareil : un diagnostic est un message du compilateur sur votre code, qu'il s'agisse d'une erreur bloquante, d'un simple avertissement, d'une note ou d'une suggestion de correction. Un bon diagnostic ne dit pas juste « non », il explique pourquoi et comment réparer.

La qualité des diagnostics est un sujet d'ingénierie à part entière, longtemps négligé puis remis au centre par Rust, Elm et Swift.

### Anatomie d'un bon message

1. **Localisation précise** : fichier, ligne, colonne, étendue (*span*) sur la portion fautive.

> **Que veut dire « span » (étendue) ?** Un *span* est la portion exacte de texte concernée par le message, du premier au dernier caractère fautif. Au lieu de dire vaguement « erreur ligne 10 », le compilateur peut souligner précisément les caractères en cause, comme un correcteur qui surligne le mot mal orthographié plutôt que la phrase entière.
2. **Cause primaire** : ce que le compilateur a vu de problématique, en termes du langage source.
3. **Contexte** : la ou les définitions concernées (où le type a été déclaré, où la variable a été *moved*).
4. **Explication** : pourquoi c'est une erreur (la règle violée).
5. **Suggestion** correctrice quand elle est mécanique : « ajoutez `&` », « importez `std::collections::HashMap` », « renommez en `foo` (typo possible) ».
6. **Exemple positif** : code qui *aurait* fonctionné, idéalement avec un *diff* applicable.

### Le standard Rust

`rustc` rend la barre haute : étendues colorées, codes d'erreur stables (`E0382`), explications longues accessibles via `--explain`, suggestions structurées que `cargo fix` peut appliquer mécaniquement, *teach mode* pour les débutants. Le secret est en grande partie une **structure de données riche** dans le compilateur : un diagnostic Rust n'est pas une chaîne, c'est un arbre annoté que l'IDE (rust-analyzer) consomme aussi.

### Récupération d'erreurs

Un parser robuste **ne s'arrête pas** à la première erreur. Stratégies classiques :

- **synchronisation** sur des tokens « stables » (point-virgule, accolade fermante) ;
- **erreurs de panic mode** : on jette les tokens jusqu'à un point de synchronisation ;
- **réparation** : on insère ou supprime un token plausible et on continue (Merr, *minimum-edit error recovery*) ;
- **îlots résilients** : Tree-sitter parse en présence d'erreurs et conserve un AST partiel, précieux pour les éditeurs.

### Avertissements et lints

> **Que veut dire « lint » ?** Un *lint* est une remarque de style ou de prudence sur du code pourtant parfaitement légal : « cette variable ne sert jamais », « cette comparaison est toujours vraie ». Le mot vient des peluches (*lint*) qu'on retire d'un vêtement : ce sont les petits défauts qu'on enlève pour faire propre, même si le vêtement est portable tel quel. Les lints sont configurables, on choisit lesquels activer.

Au-delà des erreurs, le compilateur émet des avertissements (code suspect mais légal) et des lints (règles configurables : style, motifs déconseillés, complexité). `clippy` (Rust), `clang-tidy` (C/C++), ESLint (JavaScript) sont des couches de lints construites sur l'AST ou l'IR du compilateur.

[Retour en haut de page](#table-des-matières)

---

[← Compilateurs modernes et execution](05-compilateurs-modernes-et-execution.md) · [↑ Sommaire](../README.md#table-des-matières) · [Sujets avances →](07-sujets-avances.md)
