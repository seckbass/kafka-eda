# Garantir l’ordre des messages d’un client

## La garantie Kafka

Kafka garantit l’ordre des messages **à l’intérieur d’une partition**, selon leurs offsets. Il ne garantit pas d’ordre global entre plusieurs partitions d’un topic. Pour conserver l’ordre des événements d’un client, tous ses messages doivent donc être écrits dans la même partition et traités dans l’ordre.

## 1. Utiliser l’identifiant du client comme clé

Le producteur doit renseigner la même clé, par exemple `client-42`, pour chaque message d’un même client. Le partitionneur Kafka utilise cette clé pour envoyer les messages à la même partition.

```text
clé = identifiant_client
valeur = événement du client
```

Avec le topic à trois partitions configuré dans ce projet, les messages d’un client donné seront ainsi ordonnés dans l’une des trois partitions. Les messages de clients différents peuvent être traités en parallèle. Ne pas fournir de clé (ou utiliser une clé aléatoire) ne garantit pas que les messages du client restent sur la même partition.

**Attention au nombre de partitions :** le partitionnement par hachage dépend du nombre de partitions. Augmenter ce nombre peut envoyer les nouveaux messages d’un client vers une partition différente des anciens. Pour maintenir une garantie stricte dans le temps, choisissez le nombre de partitions à l’avance et évitez de le modifier, ou concevez une stratégie de routage/migration qui préserve explicitement l’ordre par client.

## 2. Préserver l’ordre à l’écriture

- Un producteur unique doit envoyer les événements d’un client dans leur ordre logique, sans les envoyer en parallèle de façon à inverser cet ordre.
- Activer l’idempotence du producteur (`enable.idempotence=true`) évite les doublons dus aux nouvelles tentatives et préserve l’ordre des envois du producteur dans les limites de cette garantie. Kafka ajuste alors les paramètres compatibles, notamment les acquittements et les requêtes en vol.
- Attendre l’acquittement d’un envoi avant d’envoyer le suivant est une option simple lorsque l’ordre applicatif doit être rigoureusement séquentiel, au prix d’un débit inférieur.
- Plusieurs producteurs écrivant pour le même client ne partagent pas automatiquement une séquence commune. Il faut alors coordonner leur attribution et leur ordre, ou faire en sorte qu’un seul producteur traite les événements de ce client.

Exemple conceptuel de configuration :

```properties
enable.idempotence=true
acks=all
```

`acks=all` demande l’acquittement par les répliques synchronisées selon la configuration du topic ; il renforce la durabilité, mais **ne remplace pas** la clé de partition ni le traitement ordonné.

## 3. Préserver l’ordre à la lecture et au traitement

Dans un groupe de consommateurs, une partition est attribuée à un seul consommateur du groupe à la fois. C’est ce qui permet de lire séquentiellement les offsets de cette partition, tandis que d’autres consommateurs traitent les autres partitions.

Pour ne pas perdre cet ordre après la lecture :

- traiter séquentiellement les messages de chaque partition ;
- si le traitement est délégué à plusieurs threads, sérialiser les traitements par partition (ou par client) et ne pas valider un offset plus avancé tant que les offsets précédents ne sont pas terminés ;
- valider les offsets après le traitement, en respectant l’ordre des traitements. Une validation trop anticipée peut perdre des messages après une panne ; une validation désordonnée peut provoquer des retraitements ou faire sauter des messages.

Ajouter des consommateurs au-delà du nombre de partitions n’augmente pas le parallélisme de lecture de ce topic : certains consommateurs resteront inactifs. Dans ce projet, le topic par défaut a trois partitions, donc au plus trois consommateurs du même groupe peuvent lire ces partitions en parallèle.

## Ce que cela garantit — et ne garantit pas

La clé client garantit l’ordre des messages de ce client **dans Kafka et au sein de sa partition**, à condition que le producteur respecte l’ordre d’envoi et que le consommateur respecte l’ordre de traitement. Elle ne garantit pas un ordre global entre clients, entre partitions, ni entre systèmes externes appelés en parallèle. Les répliques (leader et followers) assurent la copie des données ; elles ne créent pas d’ordre supplémentaire entre partitions.

**En résumé :** clé = identifiant client, partition stable, producteur ordonné et idempotent, puis traitement séquentiel par partition et validation des offsets après traitement.
