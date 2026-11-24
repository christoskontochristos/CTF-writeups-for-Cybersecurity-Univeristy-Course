
## Writeup για το Blob Quest 1


### Source Code Analysis

Αρχικά άνοιξα το page source για να διαβάσω τον source κώδικα του παιχνιδιού ώστε να καταλάβω τι κάνει.
Κοιτάζοντας τον source κώδικα με οδήγησε να κοιτάξω τα εξής αρχεία: 
game.min.js, main.js, client.js και socket.io.js (τα οποία τα βρήκα στο web developer console, στο Debugger μέρος)
και πιο συγκεκριμένα το game.min.js το οποίο περιέχει όλη τη λειτουργικότητα του παιχνιδιού διότι ήθελα να καταλάβω τι κάνουν τα κουμπιά, όπως λέει και το ένα από τα hints. 

Κοιτάζοντας τον source κώδικα παρατηρούμε οτι είναι obfuscated, χρησιμοποιεί δηλαδή την obfuscation τεχνική _0xb173(...) για να κάνει map τα strings από έναν array (_0x173b) το οποίο το κάνει δυσκολότερο να διαβαστεί. Επομένως χρειάστηκε να χρησιμοποιήσω ένα deobfuscation tool το οποίο κάνει deobfuscate τον κώδικα σε πιο readable format. Το εργαλείο που χρησιμοποιήσα είναι το JSNice στο http://www.jsnice.org/. Απλά κάνεις copy-paste τον κώδικα στο κελί που έχει, πατάς το κουμπί NACIFY JAVASCRIPT και βγάζει την deobfuscated μορφή του. 

Κοιτάζοντας τον source κώδικα παρατήρησα τα εξής πράγματα: 

> Button Positions 

Κάθε κουμπί τοποθετείται σε ένα συγκεκριμένο location στον buttonLocs array :
```
var buttonLocs = [
  [0x4, 0xa], [0x6, 0xa], [0xd, 0xa], [0xf, 0xa],
  [0x5, 0xd], [0xe, 0xd], [0x4, 0x10], [0x6, 0x10],
  [0x8, 0x10], [0xb, 0x10], [0xd, 0x10], [0xf, 0x10]
];
```
Αυτές είναι οι θέσεις (σε hex) των κουμπιών.

> Toggling Logic

Κάθε κουμπί δηλαδή έχει δύο πιθανές καταστάσεις:
- on, value = 1
- off, value = 0
Μόλις το κουμπί γίνει pressed, η τιμή του αλλάζει από 0 σε 1.
Το updated state αποθηκεύεται στο this.buttons[index].frame
To toggling είναι rate-limited και γίνεται ανά 200ms ανά κουμπί.

> Server Communication

Μόλις γίνει toggled ένα κουμπί το στέλνεται το state των κουμπιών, δηλαδή ποιά κουμπιά είναι 
on και ποιά off στον server μέσω αυτής της συνάρτησης:
``` 
this.sendPositionThrottled = _.throttle(function () {
    var state = 0x74000;
    for (var i = 0; i < 12; i++) {
        state |= this.buttons[i].frame << i;
    }
    Client.sendPosition(-1, state);
}, 500, { trailing: false });
```
Με λίγα λόγια στέλνει στον server ένα 12-bit button state οπού το κάθε bit 
αντιπροσωπεύει και ένα κουμπί και την κατάστασή του on ή off (0 ή 1). To state των 
κουμπιών στέλνεται κάθε 500ms αν γίνει toggle

> Flag Trigger

Όπως είδαμε πιο πάνω υπάρχει επικοινωνία με τον server στην οποία στέλνεται το state των 12 κουμπιών στον server.
Δηλαδή ποιά κουμπιά είναι πατημένα. Κοιτώντας το αρχείο client.js παρατηρούμε οτι αν στείλουμε τον σωστό συνδυασμό πατημένων 
κουμπιών με την Client.sendPosition(x, y) τότε καλείται αυτή η συνάρτηση: 
```
Client.socket.on("flag-acquired", function (data) {
    Game.displayFlag(data.flag);
});
```
Δηλαδή η συνάρτηση που εκτυπώνει το flag στην οθόνη.
Με λίγα λόγια o server:
- Δέχεται ένα moved event με {x: -1, y: state}
- Ελέγχει αν το value του y είναι ο σωστός συνδυασμός 
και αν είναι σωστό κάνει socket.emit("flag-acquired", { flag: "flag{...}" })


Άρα πρέπει αν βρούμε τον σωστό συνδυασμό κουμπιών.

### Conclusion after analyzing the source code

Αφού αναλύσαμε τον πηγαίο κώδικα μπορούμε να συμπαιράνουμε οτι για να πάρουμε το flag πρέπει να λύσουμε το πρόβλημα 
δηλαδή να βρούμε τον σωστό συνδυασμό πατημένων κουμπιών. 


### Attack Plan | Exploit

Εφόσον πρέπει να βρούμε τον σωστό συνδυασμό πατημένων κουμπιών για να μπορέσουμε να ξεκλειδώσουμε το flag, μπορούμε να κάνουμε brute force όλους τους συνδυασμούς σε ένα script (θα το τρέξουμε στο web developer console) το οποίο θα στέλνει κάθε φορά το state των buttons στον server, με αυτόν τον τρόπο θα βρεθεί ο σωστός συνδυασμός και το flag θα είναι δικό μας. Επειδή τα κουμπιά είναι 12 θα χρειαστούν ακριβώς 2^12 = 4096 συνδυασμοί στη χειρότερη περίπτωση για να βρεθεί ο σωστός συνδυασμός κουμπιών.

Το Script: 


```
(function bruteForceButtons(game) {
    const max = 1 << 12;

    for (let i = 0; i < max; i++) {
        //set each button's frame to the bit in i
        for (let j = 0; j < 12; j++) {
            game.buttons[j].frame = (i >> j) & 1;
        }

        //send the state to server
        let state = 0x74000;
        for (let j = 0; j < 12; j++) {
            state |= game.buttons[j].frame << j;
        }
        Client.sendPosition(-1, state);

        console.log(`Tried combination #${i.toString(2).padStart(12, '0')}`);
    }

    console.log("Brute force finished.");
})(Game);
```

Πως λειτουργεί το παραπάνω script:

To script ελέγχει όλους τους πιθανούς συνδυασμούς πατημένων κουμπιών. Δηλαδή επειδή τα κουμπιά είναι 12 ελέγχει τους συνδυασμούς από 000000000000 μέχρι 111111111111 σε binary δηλαδή από το 0 μέχρι 4095 σε decimal και κάποιος απ αυτούς τους συνδυασμούς θα είναι οσ σωστός. Στέλνει κάθε φορά το state των κουμπιών στον server χρησιμοποιώντας τη Client.sendPosition.

To game object δίνεται σαν όρισμα στην συνάρτηση bruteForceButtons() για τον έλεγχο των κουμπιών. Η const max μεταβλητή αρχικοποιείται σε 2^12 = 4096 δηλαδή ο συνολικός αριθμός των συδυασμών που θα δοκιμαστούν. 

Στη συνέχεια με ένα for loop κάνει loop σε όλους τους πιθανούς συνδυασμούς. Δηλαδή το εξωτερικό for loop:
```
for (let i = 0; i < max; i++) 
```
οπού i είναι ο κάθε συνδυασμός. 

Για κάθε πιθανό συνδυασμό κάνει: 
```
for (let j = 0; j < 12; j++) {
            game.buttons[j].frame = (i >> j) & 1;
        }   
```
Δηλαδή για κάθε bit του συνδυασμού (12 συνολικά όπως και ο αριθμός των κουμπιών) κάνει 
(i >> j) & 1 δηλαδή κάνει isolate το j-οστό bit του i. Για παράδειγμα:

Αν i = 5 (binary 000000000101), τότε:
- j = 0: (5 >> 0) & 1 = 1
- j = 1: (5 >> 1) & 1 = 0
- j = 2: (5 >> 2) & 1 = 1

Με αυτόν τον τρόπο κάνει set την game.buttons[j].frame σε 0 ή 1 αντίστοιχα.

Αρχικοποιέι τη state μεταβλητή σε 0x74000 δηλαδή ξεκινάει από αυτό το base state το οποίο 
χρησιμοποιείται το ίδιο στον source κώδικα. 

Με το παρακάτω for loop ενώνει όλα τα 12 button states σε έναν αριθμό στη μεταβλητή state 
κάνοντας shift κάθε bit στη θέση του και στη συνέχεια το κάνει OR στη state μεταβλητή. 
```
for (let j = 0; j < 12; j++) {
            state |= game.buttons[j].frame << j;
        }
```
Τέλος στέλνει το state στον server με Client.sendPosition(-1, state) για να ελέξει αν είναι ο σωστός συνδυασμός
και να κάνει diplay το flag στην οθόνη.

Παρατήρηση: Το παραπάνω script exploit στέλνει κάθε συνδυασμό ταυτόχρονα στον server 

### How to run the exploit | Successful invocation

Τέλος για να τρέξουμε το exploit και να πάρουμε το flag ανοίγουμε την web developer console
(right click -> Inspect -> Console , στο Firefox), κάνουμε copy paste το script και πατάμε enter για να τρέξει. 

Successful invocation από web developer console: 

```
.
.
.
Tried combination #111111101110 debugger eval code:17:17
Tried combination #111111101111 debugger eval code:17:17
Tried combination #111111110000 debugger eval code:17:17
Tried combination #111111110001 debugger eval code:17:17
Tried combination #111111110010 debugger eval code:17:17
Tried combination #111111110011 debugger eval code:17:17
Tried combination #111111110100 debugger eval code:17:17
Tried combination #111111110101 debugger eval code:17:17
Tried combination #111111110110 debugger eval code:17:17
Tried combination #111111110111 debugger eval code:17:17
Tried combination #111111111000 debugger eval code:17:17
Tried combination #111111111001 debugger eval code:17:17
Tried combination #111111111010 debugger eval code:17:17
Tried combination #111111111011 debugger eval code:17:17
Tried combination #111111111100 debugger eval code:17:17
Tried combination #111111111101 debugger eval code:17:17
Tried combination #111111111110 debugger eval code:17:17
Tried combination #111111111111 debugger eval code:17:17
Brute force finished.

```

Μετά από αυτό το run , εμφανίστηκε το εξής μήνηυμα στην οθόνη: 
```
Flag: puzzle_96c1012 
```
Γράφουμε το puzzle_96c1012 στην ιστοσελίδα και επιβεβαιώνουμε οτι όντως αυτό είναι το flag. 
