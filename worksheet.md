## Worksheet
Contient les informations relative à la conception du projet

# Communication des données

Imaginons qu'on aie 12 switchs au total. 
Le but final, c'est que le côté UI doit permettre de générer la configuration de manière assez simple, et que cette config puisse être lue par l'arduino (En c++).

On a aussi un principe de banque.
Chaque banque permet de changer le comportement des switchs. 
Par exemple, je peux décider que sur la banque 1, le switch1 correspondra à C2, tandis que sur la banque 2, il correspondra à A3. 

Il nous faut donc aussi des commandes permettant de naviguer entre les banques. 

Dans l'idée, les commandes possibles sont :
- Note (C1, C#1, D1, ...)
- Prev / Next (Pour aller à la banque suivante / précédente)
- Bank (Pour aller directement sur une banque)

Si on représente ça en JSON, pour seulement 4 switchs ici, on obtient ça : 
```
{
    "bank1" : {
        "switch1" : "prev",
        "switch2" : "next",
        "switch3" : "C1",
        "switch4" : "D1"
    },
    "bank2" : {
        "switch1" : "prev",
        "switch2" : "next",
        "switch3" : "C2",
        "switch4" : "D2"
    },
    "bank3" : {
        "switch1" : "prev",
        "switch2" : "next",
        "switch3" : "C3",
        "switch4" : "D3"
    }
}
```

Ce serait facile à lire côté Arduino, donc pratique. Par contre ici on a un problème.
Il va falloir stocker cette information sur l'arduino, et la garder en mémoire tout du long, même une fois l'arduino éteinte.

Si on regarde la partie mémoire des arduinos (https://www.arduino.cc/en/tutorial/memory), il est possible de stocker cette information dans "EEPROM". Pour **l'arduino uno**, on a **1024** bytes de libre, et pour **l'arduino mega**, on en a **4096**. 


Le problème ici, c'est que si on prend la taille de notre JSON plus haut, on obtient déjà 428 byte, et ce pour seulement 3 banques de 4 switchs.

En fait, chaque caractère fait 1 byte. on est donc limité à 1024 ou 4096 caractères.

Si on minifie ce JSON, on descend maintenant à 223 bytes, ce qui est déjà mieux. 

Le but, c'est de prendre le moins de place possible finalement. Et maintenant, si plutôt que d'utiliser des **Strings** et des tableaux associatifs, on utilisait juste des tableaux de **int** ?

Imaginons :
A chaque note on associe un **int**. On veut pouvoir représenter chaque notes (**A, A#, B, C, C#, D, D#, E, F, F#, G, et enfin, G#**), sur **10 octaves** différentes (de -3 à 8). 

On a **12 notes * 10** du coup. Ce qui nous fait un total de **120 notes**.

On doit aussi pouvoir se garder quelques entiers sous le coude pour pouvoir avoir des fonctions systèmes, du type "prev", "next", "bank1", "bank2", etc ... 

Pour ces fonctions systèmes on va utiliser les 20 premiers entiers (Ce qui est suffisant je pense). 

- 0 correspond à **prev**
- 1 correspond à **next**
- 2 à 19 correspondent à banque1, banque2, banque3, etc ... au total on a 18 banques, ce qui est largement suffisant (on peut aussi voir plus loin sans soucis)

On pourrait aussi utiliser les derniers entiers (Après ceux des notes). Ca Ici, on a pas trop de contraintes, on peut grossièrement monter jusqu'à 999 (Après, on a 1000 qui contient 4 nombres et c'est plus lourd, donc chiant). 



Maintenant, si on représente tout ça dans un tableau de ints, on obtient ce genre de format :
```
[ 
    [ // Banque 1
        0,      // Prev
        1,      // Next
        57,     // | 
        58,     // |
        59,     // |
        60,     // |
        61,     // | Notes
        62,     // |
        63,     // |
        64,     // |
        65,     // |
        66      // |
    ],
    ... 
]

```

Avec 18 banques, avec uniquement des commandes "notes" égales à 140 (Le max possible), on arrive au total à **901 bytes** (Ouch, c'était limite ..). 

On peut aller encore plus loin, soyons fou.
Imaginons qu'on utilise même plus du JSON même juste un énorme entier (Un peu comme des bytes). 

On garde le principe des notes qui correspondent à un entier.

Imaginons qu'on utilise que des entiers de 3 bytes, allant de 0 à 999

- On va utiliser 000 comme séparateur (on verra pourquoi après)
- On réserve la partie 900 (Parce que c'est plus simple) :
    - 900 : prev
    - 901 : next
    - 902 -> 999 : Choix de la banque


On met tout à la suite en suivant ce format : 

DebutDeBanque - Commande1, commande2, commande2, ... , DebutDeBanque, etc ..

En gros, si on veut représenter ça :

``` 
{
    "Banque1" : {
        "Switch1" : "Next",
        "Switch2" : "Prev",
        "Switch3" : "C1",
        "Switch4" : "D1"
    }
}
```

On obtiendra ça :
``` 
{
    "Banque1" : { // 000
        "Switch1" : "Next", // 901
        "Switch2" : "Prev", // 900
        "Switch3" : "C1",   // 046
        "Switch4" : "D1"    // 048
    }
}
```

Si on met tout à la suite, on obtient ceci :
000901900046048

Maintenant, on va essayer d'aller le plus loin possible, et voyons le max qu'on peut avoir : 

- On va partir du principe qu'on a 1024 bytes max.
- Une banque fait ((12 * 3 bytes) + 3 bytes) : 12*3 pour chaque commande, et +3 pour le 000. Au total, **une banque fait 39 bytes** du coup.
- Maintenant on divise 1024 par 39 : On peut avoir au **max 26 banques** (1014 bytes). 

**26 banques** c'est le max que j'ai réussi à avoir en me remuant les méninges. Il y a peut-être des méthodes plus adaptées. 

PS : On utilise toujours 3 entiers parce que sinon ce serait quasi impossible de faire la différence entre 26 et 126 par exemple, grâce à ça on peut compter en sautant de 3 par 3. 

Si on part sur une arduino mega, on obtient un total de **105 banques**. 




**EN FAIT CORRECTION**

On a théoriquement pas besoin du "000" (seperator). Si on sait de base que chaque "commande" fait 3 bytes, alors on sait qu'une banque fait 3*12 bytes. Ca nous fait économiser de la place et on peut mettre plus de commandes du coup.

# Datasheet

### System
```
000     seperator
901     prev
902     next
903     bank1
904     bank2
905     bank3
906     bank4
907     bank5
908     bank6
909     bank7
910     bank8
911     bank9
912     bank10
913     bank11
914     bank12
915     bank13
916     bank14
917     bank15
918     bank16
919     bank17
920     bank18
921     bank19
922     bank20
923     bank21
924     bank22
925     bank23
926     bank24
927     bank25
928     bank26
```

### Notes
```
127     G9 
126     F#9
125     F9
124     E9
123     D#9
122     D9
121     C#9
120     C9
119     B8
118     A#8
117     A8
116     G#8
115     G8
114     F#8
113     F8
112     E8
111     D#8
110     D8
109     C#8
108     C8
107     B7
106     A#7
105     A7
104     G#7
103     G7
102     F#7
101     F7
100     E7
099     D#7
098 	D7
097 	C#7
096 	C7
095 	B6
094 	A#6
093 	A6
092 	G#6
091 	G6
090 	F#6
089 	F6
088 	E6
087 	D#6
086 	D6
085 	C#6
084 	C6
083 	B5
082 	A#5
081 	A5
080 	G#5
079 	G5
078 	F#5
077 	F5
076 	E5
075 	D#5
074 	D5
073 	C#5
072 	C5
071 	B4
070 	A#4
069 	A4
068 	G#4
067 	G4
066 	F#4
065 	F4
064 	E4
063 	D#4
062 	D4
061 	C#4
060 	C4
059 	B3
058 	A#3
057 	A3
056 	G#3
055 	G3
054 	F#3
053 	F3
052 	E3
051 	D#3
050 	D3 
049 	C#3
048 	C3
047 	B2
046 	A#2
045 	A2
044 	G#2
043 	G2
042 	F#2
041 	F2
040 	E2
039 	D#2
038 	D2
037 	C#2
036 	C2
035 	B1
034 	A#1
033 	A1
032 	G#1
031 	G1
030 	F#1
029 	F1
028 	E1
027 	D#1
026 	D1
025 	C#1
024     C1
023 	B0
022 	A#0
021 	A0
```


### Example data 
(112 bytes)
- Three banks
- Each bank has Prev on switch 1 and Next on switch 2
- Switchs 3 -> 12 with random midi number 
```
901902127125123121120119118117115114901902095094093092091090089087086085901902076075074073071069068067065061
```



