# Questions probables et réponses

Réponses courtes, à dire telles quelles. Deux phrases suffisent pour la plupart.

## Les chiffres à retenir

| | |
|---|---|
| Patients | 11 753, dont 944 malades, soit 8 % |
| Découpage | 9 402 pour apprendre, 2 351 gardés de côté |
| Seuil | 0,211 pour l'évaluation, 0,198 pour le modèle livré |
| Résultat final | F1 0,327, précision 0,300, rappel 0,360, AUC 0,775 |
| Fichier rendu | 2 938 patients, 325 signalés, soit 11,1 % |

## Les données

**Pourquoi remplacer les valeurs impossibles par du vide au lieu de supprimer le patient ?**

Parce qu'on a beaucoup de trous : certaines analyses manquent chez la moitié des patients. Si on supprime dès qu'il manque quelque chose, on perd les trois quarts de la cohorte. Et de toute façon, une seule analyse est fausse, pas les quarante-neuf autres.

**Comment savez-vous qu'un hématocrite à 1 est impossible ?**

L'hématocrite, c'est le pourcentage de globules rouges dans le sang. Normalement il tourne autour de 40. À 1, le patient n'aurait presque plus de sang. Et son hémoglobine est normale juste à côté, donc les deux chiffres se contredisent. C'est une erreur de l'automate.

**Et les trois autres règles ?**

Des plaquettes ou des globules blancs à zéro pile, ça n'existe pas chez quelqu'un de vivant. Un potassium au-dessus de 10, ça veut dire que le tube a été mal prélevé : les globules rouges ont éclaté et ont libéré leur potassium. On mesure le tube, pas le patient.

**Ce n'est pas tricher, de modifier les données ?**

On ne remplace pas une valeur par une autre, on la remplace par « inconnu ». On ne rajoute rien, on enlève quelque chose de faux.

**Pourquoi garder une créatine kinase à 98 801, qui a l'air aberrante ?**

Parce qu'elle est vraie : ça arrive quand un muscle est détruit. On enlève ce qui est impossible, pas ce qui est rare.

## La préparation

**Comment remplissez-vous les trous avec des formules ?**

L'automate calcule certains indices à partir d'autres mesures. Par exemple MCH égale l'hémoglobine divisée par les globules rouges, fois 10. Donc si les globules rouges manquent, on retourne la formule et on les retrouve exactement. Ça nous a récupéré 338 valeurs, avec 2 % d'erreur, alors qu'une médiane en ferait 14 %.

**Pourquoi enlever ces indices après ?**

Une fois qu'ils ont servi, ils ne disent plus rien de neuf : ce sont des copies déguisées des autres colonnes.

**Pourquoi le logarithme ?**

Parce que la plupart des analyses ont quelques valeurs géantes et beaucoup de petites. Le logarithme écrase les géantes et remet tout à la même échelle. On ne le fait que pour la régression logistique : les arbres coupent sur des seuils, ça ne change rien pour eux.

**Pourquoi `log1p` et pas `log` ?**

Parce que certaines analyses valent zéro, et le logarithme de zéro n'existe pas. `log1p` ajoute 1 avant de calculer, ce qui règle le problème.

**Pourquoi `add_indicator` ?**

Parce qu'une analyse qui manque, ça veut dire quelque chose. Quand l'indice des plaquettes est absent, les patients ont 55 plaquettes en moyenne au lieu de 209, et 86 % sont en thrombopénie au lieu de 25 %. L'automate n'a pas pu calculer l'indice, faute de plaquettes. Si on bouche le trou sans garder la trace, on perd cette information.

**Pourquoi une simple médiane, et pas mieux ?**

Parce que les méthodes plus fines devinent la valeur en regardant des patients comparables. Or ceux à qui il manque une analyse ne sont justement pas comparables aux autres. La médiane est approximative, mais honnête, et c'est l'indicateur qui porte l'information.

**Pourquoi tout mettre dans un pipeline ?**

Pour que la médiane soit calculée seulement sur les données d'apprentissage. Si on la calculait sur tout, le modèle connaîtrait déjà un peu le jeu de test. C'est ce qu'on appelle une fuite de données.

## Les modèles

**Pourquoi cinq modèles ?**

Pour voir si un modèle simple suffit, et pour avoir un point de comparaison. Et parce que le super-learner a besoin de modèles qui se trompent différemment pour servir à quelque chose.

**Pourquoi `class_weight="balanced"` ?**

Avec 8 % de malades, un modèle normal répond « pas malade » à tout le monde : il a 92 % de bonnes réponses et ne trouve personne. La pondération fait qu'un malade compte onze fois plus, donc cette stratégie ne marche plus.

**Pourquoi pas SMOTE ?**

SMOTE fabrique de faux malades en mélangeant des vrais. Chez nous, malades et non-malades se ressemblent beaucoup : fabriquer des patients dans cette zone, c'est fabriquer du bruit. La pondération donne le même résultat sans toucher aux données. On ne l'a pas testé, c'est un choix de départ.

**Pourquoi optimiser sur la précision moyenne et pas sur le F1 ?**

Parce que le F1 dépend du seuil, et qu'à ce moment-là le seuil n'est pas encore choisi. On jugerait les modèles sur une décision qu'on n'a pas encore prise.

## Le seuil

**Pourquoi régler le seuil ?**

C'est le plus gros gain du projet. Le super-learner passe de 0,078 à 0,352 de F1, juste en changeant le seuil de 0,5 à 0,211. Aucun autre choix n'a eu autant d'effet.

**Pourquoi 0,5 ne marche pas ?**

Parce qu'avec 8 % de malades, les probabilités sont presque toujours basses. À 0,5, le modèle ne signale quasiment personne.

**Pourquoi deux seuils, 0,211 et 0,198 ?**

Le premier vient du modèle testé sur les 2 351 patients mis de côté. Le second vient du modèle final, réentraîné sur tout le monde : avec plus de données, le seuil idéal bouge un peu. C'est le second qui a servi pour le fichier rendu.

**Le seuil n'est-il pas surajusté ?**

Il est choisi en validation croisée, pas directement sur les données d'entraînement. Et le test le confirme : 0,327 contre 0,352. Il reste un petit optimisme, on le dit.

## Les résultats

**Un F1 de 0,33, ce n'est pas faible ?**

Tout seul, si. Mais quand le modèle sonne l'alarme, il a raison 3 fois sur 10, contre 8 fois sur 100 au hasard. C'est presque quatre fois mieux.

**Pourquoi le test est moins bon que la validation croisée ?**

Parce que le seuil a été choisi sur les mêmes découpages qui servent à noter. Le 0,352 est donc un peu flatteur. L'écart reste petit, il n'y a pas de surapprentissage grave.

**Pourquoi pas d'intervalle de confiance ?**

Parce qu'on n'a mesuré qu'une seule fois sur le jeu de test. Ça donne un chiffre honnête, mais un seul. C'est notre principale limite.

**Pourquoi le super-learner et pas la forêt ?**

Il gagne de 0,012, alors que le résultat varie de 0,02 d'un découpage à l'autre. L'écart est donc dans le bruit. On l'a choisi, mais la forêt aurait été aussi défendable, et elle est bien plus rapide.

**Pourquoi 11,1 % de patients signalés alors qu'il y a 8 % de malades ?**

Parce que le seuil est réglé pour ne pas trop en rater. Le modèle préfère signaler un peu trop que passer à côté.

**Le glucose et le sodium en tête, c'est logique ?**

Pas vraiment, et on le dit. Ce sont des marqueurs de l'état général, pas de l'infection. Surtout, aucune variable ne domine : le signal est réparti sur une vingtaine d'analyses.

## PyTorch

**Pourquoi un réseau de neurones si les arbres gagnent ?**

Pour vérifier, pas par principe. Le réseau fait 0,288 contre 0,352. Sur des données en tableau comme les nôtres, les arbres restent très difficiles à battre.

**Pourquoi son seuil est à 0,705 ?**

Parce qu'on a dit au réseau qu'un malade compte 11,5 fois plus. Du coup il sort des probabilités plus hautes, et il faut un seuil plus haut. Les deux seuils ne se comparent pas entre eux.

**La comparaison est-elle rigoureuse ?**

Pas complètement, et on préfère le dire. Le notebook PyTorch prépare ses données avant de découper, ce qui donne une petite fuite. Ses chiffres sont donc un peu optimistes. C'est le super-learner qui a produit le fichier rendu.

## Les limites

**Quelle est votre plus grosse limite ?**

Tous ces patients avaient déjà été prélevés parce qu'on soupçonnait une infection. Il n'y a aucun patient en bonne santé. On ne sépare donc pas un malade d'un bien portant, mais une bactériémie d'une autre infection. C'est beaucoup plus dur, et aucun modèle ne réglera ça avec ces données.

**Est-ce utilisable à l'hôpital ?**

Pas en l'état. Il faudrait le tester dans un autre hôpital. Et surtout, rater un malade est plus grave que déclencher une hémoculture inutile, alors que notre mesure traite les deux erreurs pareil.

**Qu'est-ce que vous feriez avec plus de temps ?**

Un intervalle de confiance, pour dire à quel point notre chiffre est fiable. Et relancer la recherche des réglages sur la version finale des données.
