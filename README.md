# LAB 11 : Bypass de la Détection de Root Android avec Frida (Hooks Java & Natif)

Objectif du lab
Ce lab a pour objectif de comprendre comment les applications Android détectent le root et comment contourner ces mécanismes en utilisant Frida. L’analyse se fait à deux niveaux : Java et natif (C/C++).

1. Installation et preuve (20 pts)

Afin de vérifier le bon fonctionnement de l’environnement, plusieurs commandes ont été exécutées.

frida --version permet de vérifier que Frida est correctement installé.
python -c "import frida; print(frida.__version__)" confirme que le module Python Frida est fonctionnel.
adb devices permet de vérifier que l’appareil Android est bien connecté.

<img width="945" height="199" alt="image" src="https://github.com/user-attachments/assets/81fe88b1-6ef7-4b6e-86e2-72391002a5bc" />

Ces commandes montrent que l’environnement est prêt pour l’analyse dynamique.

2. Déploiement et visibilité (30 pts)

Le service frida-server a été lancé sur l’appareil Android avec les privilèges root :

adb shell
su
/data/local/tmp/frida-server -l 0.0.0.0

<img width="945" height="94" alt="image" src="https://github.com/user-attachments/assets/0a7a75c3-0c2b-4c0b-a989-34a2c7ced26b" />

Ensuite, la commande suivante a été utilisée pour lister les applications :

frida-ps -Uai

<img width="945" height="570" alt="image" src="https://github.com/user-attachments/assets/cff21ee6-5c94-44e8-b6e6-37bd058857ce" />

Cette commande a permis d’afficher plusieurs applications installées sur l’appareil, confirmant que Frida communique correctement avec le système Android.

3. Bypass Java (30 pts)

Dans cette étape, nous avons contourné la détection de root au niveau Java en utilisant le script bypass_root.js.

✔️ Principe

Le script intercepte plusieurs fonctions sensibles :

Build.TAGS → forcé à release-keys
File.exists → masque les fichiers su
Runtime.exec → bloque les commandes liées au root
Après injection du script avec Frida :

frida -U -f owasp.mstg.uncrackable1 -l bypass_root.js

Les logs suivants ont été observés :

<img width="945" height="460" alt="image" src="https://github.com/user-attachments/assets/9156a58e-cd50-4d7d-8810-fe6d3f7f78f1" />
Ces messages confirment que les vérifications de root ont été interceptées avec succès.

4. Natif / Trace (20 pts)

Dans cette étape, nous avons analysé les appels natifs utilisés par l’application.

✔️ Identification avec frida-trace
frida-trace -U -f owasp.mstg.uncrackable1 -i open -i access -i stat
<img width="945" height="463" alt="image" src="https://github.com/user-attachments/assets/b8b4609e-50c1-492a-b7fe-57ed9eb9365e" />

Les fonctions suivantes ont été identifiées :

open
access
stat

Ces fonctions sont utilisées pour vérifier l’existence de fichiers liés au root.

✔️ Adaptation du script natif

Le script bypass_native.js a été utilisé pour intercepter ces appels.

Modification principale :

utilisation de getExport() pour compatibilité avec Frida 17
ajout d’un test pour valider les hooks
Voici la partie ajoutée
•	// Test pour afficher un log Blocked
•	setTimeout(function () {
•	  console.log('[*] Native self-test started');
•	
•	  try {
•	    const accessPtr = getExport('access');
•	
•	    if (!accessPtr) {
•	      console.log('[-] access export not found');
•	      return;
•	    }
•	
•	    const access = new NativeFunction(accessPtr, 'int', ['pointer', 'int']);
•	    const path = Memory.allocUtf8String('/system/bin/su');
•	
•	    access(path, 0);
•	
•	  } catch (e) {
•	    console.log('[-] self-test failed:', e);
•	  }
•	}, 1000);

✔️ Résultat

<img width="945" height="415" alt="image" src="https://github.com/user-attachments/assets/f9c06976-eac4-46da-8866-9c3f67ba8e64" />

L’application s’exécute normalement, ce qui prouve que le bypass Java fonctionne.

