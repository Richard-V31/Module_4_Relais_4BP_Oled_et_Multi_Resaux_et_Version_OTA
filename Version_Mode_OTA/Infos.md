/*
Pour rendre ça flexible, il faut refactoriser vers une architecture basée sur un tableau de programmateurs plutôt que des variables séparées PR1/PR2/PR3/PR4 :

Une structure Programmateur { pin, heureDebut, heureFin, modeAuto, relayState, nom }
Un tableau Programmateur programmateurs[N] où N est défini par la configuration
Des routes web génériques (ex: /toggle-mode?id=2) au lieu d'une route par relais
La page HTML JS génère déjà dynamiquement ses lignes depuis un tableau RELAYS — donc côté affichage c'est presque prêt, il « suffit » d'adapter le nombre d'entrées
La sauvegarde NVS en boucle sur le tableau au lieu de champs nommés PR1/PR2/PR3/PR4

  { "PR1", "Programmation 1", "Cuisine",  "#f59e0b", 32, "08:00", "18:00", true, false },
//  👉🚩Champs : { id, nom affiché, sous-titre, couleur (hex), broche GPIO,
//              heure de début par défaut, heure de fin par défaut,
//              mode auto par défaut, état par défaut }

Un tableau de configuration unique, avec des routes web génériques. Ajouter un relais deviendra une simple ligne à copier-coller dans le tableau, 
sans toucher au reste du code.

Un seul endroit à modifier : le tableau programmateurs[] . Chaque ligne = un relais (id, nom, couleur, broche GPIO, horaires par défaut). 
Pour passer de 4 à 8 relais, il suffit d'ajouter 4 lignes — rien d'autre à toucher.
 Une seule route gère n'importe quel nombre de relais.
Page web adaptative : le JavaScript interroge une nouvelle route /get-config au chargement pour savoir combien de relais existent 
et comment les afficher — la mise en page n'est plus figée dans le HTML.
Sauvegarde NVS en boucle : les clés sont générées dynamiquement (PR1debut, PR2debut...) au lieu d'être écrites une par une.
Écran OLED en pages tournantes : comme il ne peut afficher que 5 relais à la fois, l'écran bascule automatiquement toutes les 4 secondes 
vers la page suivante si vous dépassez 5 relais.

🚨 Pour aller au-delà de 4 relais :
L'ESP32 classique offre une quinzaine de broches GPIO utilisables en sortie (j'ai mis la liste en commentaire dans le code) — au-delà, il faudrait un 
module d'extension I2C (PCF8574).

  { "PR1", "Programmation 1", "Cuisine",  "#f59e0b", 32, "08:00", "18:00", true, false },
#f59e0b est un code couleur hexadécimal, le même format que ceux utilisés en CSS/HTML pour définir une couleur.

Décomposition :

# indique que ce qui suit est un code couleur hexadécimal
06 → composante rouge (0 à 255, ici 6 en décimal = très faible)
b6 → composante verte (182 en décimal = assez forte)
d4 → composante bleue (212 en décimal = forte)

Chaque paire de caractères est un nombre en base 16 (hexadécimal), de 00 (aucune intensité) à ff (intensité maximale).
f59e0b donne donc un mélange peu de rouge + beaucoup de vert + beaucoup de bleu, ce qui produit un cyan/turquoise :
Composantes RVB de #f59e0b
Intensité (0-255)
Rouge
Vert
Bleu

Dans le code, c'est simplement la couleur d'accent attribuée au relais "PR2 / Portail" : elle sert à colorer le liseré à gauche de sa carte dans l'interface 
web (border-left:3px solid var(--accent)), ainsi que certains effets visuels au survol/appui du bouton "Forcer" (via --accent-rgb, qui est calculé 
automatiquement en JS avec la fonction hex2rgb()).

On peut remplacer cette valeur par n'importe quel code hexadécimal pour changer la couleur associée à ce relais — par exemple 
#ec4899 pour du rose, ou 
#22c55e pour du vert.

//-----------------------------------------------------------//
/*
🚩1️⃣ Bouton Auto/Manuel
1. CSS ( lignes 231-240 )

  Bouton rectangulaire AUTO / MANUEL  
.mode-btn{
  min-width:92px; padding:8px 12px; border:none; cursor:pointer;
  border-radius:12px; /* 🚨bords arrondis -> ajuster ce rayon 
  font-size:.85rem; font-weight:900; letter-spacing:.04em; text-transform:uppercase; color:#fff;
  text-align:center; transition:background .2s, box-shadow .2s;
}
/* 🚨Couleur fond mode AUTO Orange*/
.mode-btn.auto{
  background:rgba(251, 195, 12, 1);
  border:5px solid rgba(134, 119, 79, 1);
}
/* 🚨Couleur fond mode MANUEL Vert */
.mode-btn.manuel{
  background:rgba(19, 151, 155, 1); 
  border: 5px solid rgba(17, 121, 125, 1);
  }

Ce code CSS sert à styliser un élément HTML qui possède à la fois la classe .mode-btn et la classe .manuel.
Voici l'explication détaillée des deux règles appliquées :
1. background: rgba(19, 151, 155, 1);
Cette ligne définit la couleur de fond de l'élément.rgba signifie Rouge, Vert, Bleu, et Alpha (l'opacité).
Les trois premiers chiffres (19, 151, 155) créent une couleur bleu-vert / turquoise.Le dernier chiffre 1 indique une opacité maximale (100%), 
ce qui signifie que la couleur est complètement opaque (pas de transparence).

2. border: 5px solid rgba(15, 65, 73, 0.8);
Cette ligne ajoute une bordure tout autour de l'élément.5px détermine l'épaisseur de la 
bordure (5 pixels).solid indique que la bordure est une ligne continue (pas de pointillés ni de tirets).rgba(15, 65, 73, 0.8) 
définit la couleur de cette bordure, qui est un bleu-vert très foncé. Le 0.8 signifie que la bordure a une opacité de 80%,
 elle laisse donc légèrement transparaître ce qui se trouve derrière elle.

pour les couleurs  https://rgbacolorpicker.com/

2. HTML (lignes 367-373, dans initRelays())

html
<div>
  <button type="button" class="mode-btn" id="${r.id}mode-btn" onclick="toggleMode('${r.id}')">---</button>
</div>

3. JavaScript ( lignes 400-401, dans update())

javascript
const modeBtn = document.getElementById(p + 'mode-btn');
modeBtn.innerText = d.auto ? "AUTO" : "MANUEL";
modeBtn.className = "mode-btn " + (d.auto ? "auto" : "manuel");

Rien d'autre n'a été touché : ni le back-end ESP32 (routes /toggle-mode, /get-data, etc.), ni le reste de l'interface.

.mode-btn.manuel{background:rgba(39, 245, 184, 1); box-shadow:0 0 12px rgba(39, 245, 184, 0.82);}
1. .mode-btn.manuel — le sélecteur CSS
Ça cible tous les éléments qui ont à la fois la classe mode-btn et la classe manuel (les deux classes collées sans espace = "ET", pas "OU"). 
C'est exactement ce que fait le JavaScript quand d.auto est false : modeBtn.className = "mode-btn manuel".

2. background:rgba(39, 245, 184, 1); — la couleur de fond du bouton
rgba(...) = un modèle de couleur à 4 valeurs
39 = quantité de Rouge (0 à 255)
245 = quantité de Vert (0 à 255)
184 = quantité de Bleu (0 à 255)
Ces trois valeurs donnent une teinte turquoise/vert d'eau (beaucoup de vert, pas mal de bleu, très peu de rouge)
1 = le canal Alpha (opacité), où 1 = 100% opaque (couleur pleine, on ne voit rien à travers)

3. box-shadow:0 0 12px rgba(39, 245, 184, 0.82); — l'ombre portée (le halo lumineux)
0 (1er) = décalage horizontal → 0 = pas de décalage sur les côtés
0 (2e) = décalage vertical → 0 = pas de décalage haut/bas
12px = le flou (blur radius) → plus le chiffre est grand, plus le halo est diffus/large
rgba(39, 245, 184, 0.82) = même couleur turquoise que le fond, mais avec 0.82 d'opacité (82%), donc légèrement transparente 
pour créer un effet de lueur qui se fond dans le fond sombre plutôt qu'un contour dur

//-------------------------------------------------
🚩2️⃣Code modifié pour le Switch Auto/Manuel

Ligne 369 — dans la construction HTML de chaque ligne de relais :
// Avant
<input type="checkbox" id="${r.id}auto-switch" onclick="fetch('/toggle-mode?id=${r.id}')">
// Après
<input type="checkbox" id="${r.id}auto-switch" onclick="toggleMode('${r.id}')">

Après la ligne 419 (juste après la fin de la fonction update(), avant saveRelay()) — ajout d'une nouvelle fonction :
async function toggleMode(id) {
  await fetch('/toggle-mode?id=' + id);
  update();
}*/

//-------------------------------------------------
🚩 3️⃣Ce que fait ce nouveau code :
Évite les faux démarrages : La boucle while bloque le programme pendant un court instant 
(généralement entre 1 et 3 secondes) jusqu'à ce que getLocalTime() renvoie une heure réelle.
Retour visuel : L'écran OLED affiche "Synchro heure Internet..." pour informer l'utilisateur 
de ce qui se passe.Sécurité (Timeout) : Si votre box internet est en panne ou que les serveurs 
NTP ne répondent pas, la boucle s'arrête d'elle-même après 10 secondes (tentative < 20) pour que l'ESP32 
démarre quand même son serveur web, vous permettant ainsi d'y accéder en local.

//-------------------------------------------------
// 🚩4️⃣  Commutation Réseau
1. Liste noire temporaire + reconnexion propre — lignes
 567–632 (fonction connectToBestNetwork())
 567-568 : nouvelles variables lastFailTime[] et BLACKLIST_DURATION
 577 : WiFi.disconnect(true) avant le scan
 598 : vérification blacklisted dans la sélection du meilleur réseau
fin de fonction (~ligne 620-630) : mise à jour de lastFailTime selon succès/échec

2. Retour automatique vers le meilleur réseau — lignes 
 637–685 (nouvelle fonction checkForBetterNetwork())
 645-646 : constantes BEST_NETWORK_RECHECK_INTERVAL (60 s) et RSSI_SWITCH_MARGIN (8 dB)
 648 : début de la fonction

3. Désactivation de la reconnexion auto interne — ligne 
831 (dans setup())
WiFi.setAutoReconnect(false);

4. Appel périodique dans la boucle principale — lignes 
 1064–1067 (dans loop())
appel de checkForBetterNetwork() toutes les 60 s


/// Roujout du % a coté de WiFi
Voici l'ensemble des ajouts, dans l'ordre du fichier :

CSS (lignes 313-321) — style du badge et du regroupement avec le bouton :

css
.wifi-indicator{display:flex; align-items:center; gap:6px; flex-shrink:0;}
.wifi-pct{font-size:.7rem; font-weight:700; color:var(--sub); min-width:2.6em; text-align:right;}
.wifi-pct.good{color:var(--ok);}
.wifi-pct.mid{color:#fbbf24;}
.wifi-pct.weak{color:#fb7185;}

HTML (ligne 362-373) — le bouton wifi est désormais enveloppé dans <div class="wifi-indicator">, avec le badge juste avant :

html
<div class="wifi-indicator">
  <span class="wifi-pct" id="wifi-pct">--%</span>
  <button class="info-btn" onclick="openInfo()" title="Infos système">
    ...(svg inchangé)...
  </button>
</div>

JavaScript (lignes 475-489) — fonction qui met à jour le badge et sa couleur :

js
function updateWifiPct(pct) {
  const el = document.getElementById('wifi-pct');
  if (!el) return;
  if (typeof pct !== 'number' || pct < 0) {
    el.textContent = '--%';
    el.className = 'wifi-pct';
    return;
  }
  el.textContent = pct + '%';
  el.className = 'wifi-pct ' + (pct >= 67 ? 'good' : pct >= 34 ? 'mid' : 'weak');
}

JavaScript (ligne 496) — appel de cette fonction à chaque cycle de update() :

js
updateWifiPct(data.wifiPct);

JavaScript (lignes 596-598) — le popup "Infos système" affiche désormais dBm + % :

js
document.getElementById('info-rssi').innerText =
  info.rssiPct >= 0 ? `${info.rssi} (${info.rssiPct}%)` : info.rssi;

C++ (lignes 817-823) — la fonction de conversion dBm → % côté ESP32 :

cpp
int rssiToPercent(int rssiDbm) {
  if (rssiDbm <= -100) return 0;
  if (rssiDbm >= -50) return 100;
  return 2 * (rssiDbm + 100);
}

C++ (ligne 1162) — ajout du champ dans la route /get-data (interrogée chaque seconde) :

cpp
doc["wifiPct"] = (WiFi.status() == WL_CONNECTED) ? rssiToPercent(WiFi.RSSI()) : -1;

C++ (ligne 1182) — ajout du même champ dans la route /get-info (pour le popup) :

cpp
doc["rssiPct"] = connected ? rssiToPercent(WiFi.RSSI()) : -1;


//********* Mode OTA
Nouvel include — #include <ArduinoOTA.h>, juste après ESPmDNS.h.

Nouvelle fonction setupOTA(), appelée une fois dans setup() (après la connexion WiFi et le mDNS, dont l'OTA a besoin) :

définit le hostname OTA (le même que le mDNS, ex. richardv) ;
active un mot de passe OTA si vous définissez SECRET_OTA_PASSWORD dans arduino_secrets.h (ex. #define SECRET_OTA_PASSWORD "votre_mot_de_passe") — fortement recommandé, sinon n'importe quel appareil du réseau pourrait reflasher l'ESP32 ; sans ce define, un message d'avertissement s'affiche au démarrage sur le port série mais le programme fonctionne quand même ;
affiche la progression du transfert sur l'écran OLED (s'il est présent) : "Mise à jour OTA...", le pourcentage, puis "Mise à jour OK ! Redémarrage..." — utile puisque l'ESP32 sera dans son boîtier, sans écran d'ordinateur à côté ;
log les erreurs éventuelles sur le port série (échec d'authentification, de connexion, etc.).

Dans loop() : ajout de ArduinoOTA.handle(); juste après checkPhysicalButtons();, pour que l'ESP32 écoute en permanence les demandes de mise à jour sans bloquer le reste (boutons, relais, serveur web).

À l'usage : après avoir flashé une première fois par USB avec cette version, la carte apparaîtra comme un port réseau dans l'IDE Arduino (menu Outils > Port), tant qu'elle reste sur le même WiFi — vous pourrez alors téléverser sans câble.
//***** Utilisation OTAVoici comment utiliser la mise à jour OTA une fois le nouveau programme flashé sur l'ESP32 :

1. Premier flash obligatoire par USB

La toute première fois, vous devez téléverser cette version par câble USB comme d'habitude. C'est cette version qui contient le code OTA lui-même — l'ESP32 ne peut pas "apprendre" l'OTA à distance, il faut qu'il tourne déjà avec ce code.

2. Vérifier que l'ESP32 apparaît sur le réseau

Après ce premier flash, ouvrez le moniteur série (115200 bauds) : vous devez voir le message OTA pret : mise a jour possible via le WiFi depuis l'IDE Arduino. Cela confirme que l'ESP32 est bien connecté au WiFi et écoute les demandes OTA.

3. Sélectionner le port réseau dans l'IDE Arduino

Dans l'IDE Arduino, allez dans Outils > Port. En plus des ports USB habituels (COM3, etc.), vous devriez voir apparaître une entrée réseau du type :

richardv at 192.168.x.x (ESP32)

Sélectionnez ce port réseau (au lieu du port USB).

Remarque : cette entrée peut prendre quelques secondes à apparaître après le démarrage de l'ESP32 — le temps que le WiFi et l'OTA s'initialisent.

4. Téléverser normalement

Faites vos modifications dans le code, puis cliquez sur le bouton Téléverser (flèche) comme d'habitude. L'IDE va :

compiler le programme,
l'envoyer par WiFi vers l'ESP32,
si SECRET_OTA_PASSWORD est défini, l'IDE vous demandera le mot de passe (une fenêtre s'ouvre automatiquement) — pas besoin de le retaper à chaque fois si vous cochez l'option pour le mémoriser.

5. Suivre la progression

Pendant le transfert, l'écran OLED (si présent) affiche le pourcentage d'avancement, puis "Mise à jour OK ! Redémarrage...". L'ESP32 redémarre automatiquement une fois le nouveau programme installé — comme après un flash USB classique.

Points importants

Si vous n'avez pas défini SECRET_OTA_PASSWORD dans arduino_secrets.h, n'importe quel appareil du réseau WiFi pourrait en théorie reflasher l'ESP32. Ajoutez-le si ce n'est pas déjà fait :
cpp
  #define SECRET_OTA_PASSWORD "votre_mot_de_passe"
L'ESP32 doit rester sur le même réseau WiFi que votre ordinateur pour que l'OTA fonctionne (le port réseau n'apparaît pas s'ils ne sont pas sur le même sous-réseau).
Si l'OTA échoue en cours de route, l'ESP32 redémarre généralement avec l'ancien programme intact (il ne reste pas "bloqué" à mi-transfert) — mais gardez un accès USB de secours au cas où.


//********🚨⌚️  Programmation horaireLe contexte
      if (p.heureDebut < p.heureFin) newState = (now >= p.heureDebut && now < p.heureFin);
        else newState = (now >= p.heureDebut || now < p.heureFin);

now, p.heureDebut et p.heureFin sont tous les trois des chaînes de caractères au format "HH:MM" (par exemple "20:00", "06:00", "23:47"). 
L'astuce de tout ce code, c'est que comparer ces chaînes avec <, >=, etc. revient à comparer des heures, à condition que le format soit 
toujours sur 2 chiffres pour les heures et 2 pour les minutes (ce que garantit l'input <input type="time"> de la page web). 
Concrètement, "09:00" < "10:00" est vrai, "20:00" > "06:00" est vrai, exactement comme si on comparait des nombres.

Pourquoi deux cas différents ?
Le code doit gérer deux types de plages horaires :

une plage qui reste dans la même journée (ex: 08:00 → 18:00)
une plage qui traverse minuit (ex: 20:00 → 06:00)

Le test if (p.heureDebut < p.heureFin) sert justement à distinguer les deux : si l'heure de début est plus petite que l'heure de fin, 
c'est une plage "normale" (ne traverse pas minuit). Sinon (début >= fin), on considère que la plage traverse minuit.

---Cas 1 — plage normale : now >= heureDebut && now < heureFin
Ici c'est un simple encadrement : le relais est ON si l'heure actuelle est comprise entre le début (inclus) et la fin (exclue). 
Exemple avec 08:00 → 18:00 :
à 07:59 → now >= "08:00" est faux → OFF
à 08:00 → les deux conditions sont vraies → ON
à 17:59 → toujours ON
à 18:00 → now < "18:00" devient faux → OFF

Un simple AND suffit parce que la zone "ON" est un seul bloc continu dans la journée.

---Cas 2 — plage qui traverse minuit : now >= heureDebut || now < heureFin
Ici, la zone "ON" n'est plus un seul bloc continu entre 00:00 et 23:59 : elle est coupée en deux morceaux par le passage à minuit. 
Avec 20:00 → 06:00, la zone ON c'est :
de 20:00 à 23:59 (un premier morceau, en fin de journée)
de 00:00 à 05:59 (un deuxième morceau, en début de journée suivante)

Un AND ne fonctionnerait pas ici : il n'existe aucune heure qui soit à la fois >= 20:00 ET < 06:00 
(c'est mathématiquement impossible dans la même journée), donc avec un AND le relais ne s'allumerait jamais. C'est pour ça qu'on utilise
 un OR : le relais est ON si l'heure actuelle appartient à l'UN OU L'AUTRE des deux morceaux :

now >= "20:00" → couvre le morceau du soir (20:00 à 23:59)
now < "06:00" → couvre le morceau du matin (00:00 à 05:59, puisque juste après minuit now redevient petit, genre "00:15", "01:30"...)

Exemple pas à pas avec 20:00 → 06:00 :

à 19:59 → now >= "20:00" faux, now < "06:00" faux → OFF
à 20:00 → now >= "20:00" vrai → ON (peu importe l'autre condition, OR suffit)
à 23:59 → toujours vrai via la première condition → ON
à 00:00 → la première condition devient fausse ("00:00" >= "20:00" faux), mais la deuxième devient vraie ("00:00" < "06:00" vrai) → ON (le relais reste allumé sans coupure)
à 05:59 → toujours ON via la deuxième condition
à 06:00 → now >= "20:00" faux ET now < "06:00" faux (égalité, pas d'infériorité stricte) → OFF

C'est ce mécanisme qui permet au relais de rester allumé en continu de 20h00 le soir jusqu'à 6h00 le lendemain matin, sans "trou" ni fausse coupure à minuit : le passage de 23:59 à 00:00 fait juste basculer quelle moitié du OR est vraie, mais le résultat global reste ON tout du long.



*/