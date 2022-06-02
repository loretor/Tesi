# Progetto di Tesi con Raspberry
Creazione di una serra automatica con la scheda Raspberry

# Componenti del gruppo di lavoro
- Matteo Carminati
- Lorenzo Torri

# Organizzazione della repository
Nella cartella [Modelli](/Modelli) sono presenti le rappresentazioni mediante UML del progetto. Per ora l'unica rappresentazione creata è quella di uno StateChart Diagram.

Nella cartella [Codice](/Codice) sono presenti tutti i file .py creati per controllare sensori e attuatori tramite Raspberry. In particolare sono presenti i singoli file python che sono stati usati per controllare singolarmente i sensori e per chiudere ed aprire il circuito che collega rasperry alla scheda relais.

Il file [controller.py](Codice/controller.py) è la prima versione di codice python che verrà utilizzato per poter controllare raspberry e poter permettere alla scheda di svolgere più attività e controllare lo scheduling di queste ultime.

# Spiegazione controller.py
Il file è ancora abbastanza confuso; in futuro l'idea è quella di suddividere ogni classe in un file python proprio, di modo tale poi da avere una migliore organizzazione del progetto. 
Il flow di controllo del rapsberry è fortemente influenzato dallo StateChart presente nella cartella [Modelli](/Modelli).
Per ora siamo riusciti a implementare tramite codice solamente la parte interna allo stato "Luce accesa", focalizzandoci sullo scheduling delle due diverse attività di lettura del sensore DTH22 (per la temperatura e umidità dell'aria) e del sensore Capacitive Soil Moisture (per la temperatura del suolo).

Queste due attività sono state implementate con dei thread che condividono un lock per la risorsa condivisa. Ogni attività di lettura dei due sensori si basa su una lista di letture. In particolare ognuno dei due thread ha associato una queue di valori, tale per cui ogni volta che il sensore legge un valore lo inserisce nella sua queue. Se la queue è piena si elimina il valore più vecchio che è stato letto dal sensore e si aggiunge il nuovo (politica FIFO). 
E' stata scelta questa gestione dei valori letti da un sensore per rendere statisticamente più valide le misurazioni. Infatti può accadere (a volte data la scarsa qualità di alcuni sensori) che ci possano essere delle letture errate o che non catturano i veri dati della realtà. Per rendere quindi tali misurazioni il meno impattanti possibili sul controllore, è necessario aumentare il numero di campioni e lavorare sempre sulla media di questi ultimi.
Questa è la porzione di codice del controllore che gestisce l'aggiunta di un nuovo elemento alla lista con l'aggiornamento della media di quest'ultima.
```
#metodo per aggiornare il valore di una media e la lista degli ultimi valori letti da un sensore
def update_M(lista, M, valore):
    media = M
    #se la lista è piena bisogna togliere il più vecchio valore letto (politica FIFO) e aggiornare la media
    if lista.full():
        y = lista.get()
        media = ((lista.qsize()+1) * media - y) / (lista.qsize())

    #se la lista non è piena o è terminato l'if basta aggiungere il valore e aggiornare la media
    lista.put(valore)
    if (lista.qsize() == 0):
        media = valore
    else:
        media = ((lista.qsize() - 1) * media + valore) / lista.qsize()

    return media
```

Per ultimo nel codice sono presenti le queue per i due sensori e le loro medie. Inoltre sono presenti anche altre variabili necessarie per fare i controlli per eventualmente azionare la scheda relais. Per ora la grandezza di queste pile è di solo 5 perchè nelle nostre fasi di testing era necessario controllare più velocemente il corretto funzionamento del Rapsberry. In futuro la grandezza delle liste sarà molto più grande.
```
Listdimension = 5

#liste per fare una media di un certo numero di valori e non considerare solo il singolo
lista_valori_dht22_temperatura = queue.Queue(Listdimension)
lista_valori_dht22_umidita = queue.Queue(Listdimension)
lista_valori_capacitive = queue.Queue(Listdimension)

#valori di riferimento per prendere decisioni
M_temperatura_aria = 0
M_umidita_aria = 0
M_umidita_suolo = 0

# valori costanti
Max_umidita_aria = 60
Min_umidita_aria = 50
Min_umidita_suolo = 40
countIrrigazioni = 0
# dopo quante irrigazioni bisogna fertilizzare
numeroIrrigazioni = 2
```

La gestione dei thread è abbastanza classica, infatti seguendo lo Statechart abbiamo creato due diversi thread con una classe build in di Python con associate l'attività che il thread deve svolgere.
Spieghiamo prima come sono creati i thread e poi come abbiamo gestito il loro scheduling.
I thread hanno associato una funzione che rappresenta il loro task da svolgere. La funzione sia nel caso di DHT22 che nel caso del Capacitive Soil Moisture Sensor consiste:
- porzione di codice dove si setta il collegamento tra raspberry e il sensore per ottenere il dato (è uno dei file .py presenti nella cartella Codici che abbiamo integrato in questa funzione specifica.
- aggiorna i valori della prorpia queue con il metodo visto in precedenza
- sulla base dei valori ottenuti fa dei controlli e in caso stampa a video quello che dovrebbe fare (per ora non abbiamo integrato al controller.py tutta la parte di gestione della scheda realais che abbiamo sui codici singoli presenti in Modelli, in quanto abbiamo voluto effettuare delle fasi di testing per controllare che il lavoro dei thread funzioni correttamente).
- se si entra in uno dei if (quindi verrebbe teoricamente acceso un attuatore) si eliminano tutti i valori all'interno della propria queue.

Riportiamo ad esempio la funzione del DTH22, per il capacitive è simile, la si trova nel codice come def activity_Capacitive():
```
## funzione attività del sensore DTH22, per misura umidità aria e temperatura
def activity_DHT22():
    global M_temperatura_aria
    global M_umidita_aria
    
#     lettura valori
    DHT = 21
    h,t = dht.read_retry(dht.DHT22, DHT)

    print('DHT22')
    #aggiorno la lista_valori_dht22_temperatura e la M_temperatura_aria
#     t = random.randint(0,22)
    M_temperatura_aria = update_M(lista_valori_dht22_temperatura, M_temperatura_aria, t)
    M_umidita_aria = update_M(lista_valori_dht22_umidita, M_umidita_aria, h)
    
#     controllo attività
#   le attività possono essere svolte solo se l'array dei valori è pieno
    if(lista_valori_dht22_umidita.full()):
#       Umidificazione
        if M_umidita_aria < Min_umidita_aria:
            activitycaso("umidificazione") 
            time.sleep(10)
#             azzeriamo le medie e svuotiamo gli array
            clearList(lista_valori_dht22_temperatura)
            clearList(lista_valori_dht22_umidita)
            M_temperatura_aria = 0
            M_umidita_aria = 0
#        Ventilazione Alta
        elif M_umidita_aria > Max_umidita_aria:
            activitycaso("ventilazione alta")
            time.sleep(10)
            #             azzeriamo le medie e svuotiamo gli array
            clearList(lista_valori_dht22_temperatura)
            clearList(lista_valori_dht22_umidita)
            clearList(lista_valori_capacitive)
            M_temperatura_aria = 0
            M_umidita_aria = 0
            M_umidita_suolo = 0
         
    print(list(lista_valori_dht22_umidita.queue))
    print("M: "+str(M_umidita_aria))
    print("--------------------")
    print(list(lista_valori_dht22_temperatura.queue))
    print("M: "+str(M_temperatura_aria))
    print("--------------------")
```



L'idea è infatti quella di alternare la lettura prima di un sensore (thread1) e poi quella del secondo sensore (thread2), di modo tale che poi sempre nel momento in cui ogni thread possiede il lock possa eventualmente azionare degli attuatori qualora sulla base della propria lettura sia necessario intervenire sulla serra (esempio: DHT22 legge una nuova temperatura, calcola la media delle ultime temperature lette e osserva che la temperatura media è troppo alta per la serra, per cui chiude il circuito della scheda relais e attiva la ventola per poter permettere un recircolo d'aria; una volta terminato rilascia il lock per permettere al Capacitive di proseguire con la sua lettura).

