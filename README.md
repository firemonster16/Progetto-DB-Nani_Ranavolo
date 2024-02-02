***TABELLE***

**- UTENTE:**
```SQL
CREATE TABLE IF NOT EXISTS UTENTE
(
    idUtente SERIAL,
    login varchar(25) UNIQUE,
    password varchar(25),
    nome varchar(30),
    cognome varchar(30),
    dataNascita Date,
    email varchar(50) check(email LIKE '%@%.%'),
    ruolo char(6) DEFAULT 'Utente',
    CONSTRAINT pk_utente PRIMARY KEY(idUtente)
)
```
**- PAGINA:**
```SQL
CREATE TABLE Pagina
(
    idPagina SERIAL,
    titolo varchar(80),
    dataOraCreazione TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    idAutore int NOT NULL,
    constraint pk_pagina PRIMARY KEY(idPagina),
    constraint Fk_utente FOREIGN KEY(idAutore) REFERENCES utente(idUtente)
)
```
**- VISIONA:**
```SQL
CREATE TABLE Visiona
(
    idUtente int,
    idPagina int,
    dataVisone DATE DEFAULT CURRENT_DATE,
    oraVisione TIME DEFAULT CURRENT_TIME,
    CONSTRAINT Fk_Utente FOREIGN KEY(idUtente) REFERENCES UTENTE(idUtente),
    CONSTRAINT Fk_pagina FOREIGN KEY(idPagina) REFERENCES PAGINA(idpagina)
)
```
**- FRASE CORRENTE**
```SQL
CREATE TABLE fraseCorrente
(
    StringaInserita varchar(1000),
    numerazione int,
    dataInserimento DATE DEFAULT CURRENT_DATE,
    oraInserimento TIME DEFAULT CURRENT_TIME,
    idPagina int,
    collegamento pk_pagina,
    CONSTRAINT pk_frase PRIMARY KEY(StringaInserita, numerazione, idPagina),
    CONSTRAINT fk_pagina FOREIGN KEY(idPagina) REFERENCES PAGINA(idPagina),
    CONSTRAINT fk_collegamento FOREIGN KEY(collegamento) REFERENCES PAGINA(idPagina)
)
```
**- MODIFICA PROPOSTA**
```SQL
CREATE TABLE ModificaProposta
(
    idModifica Serial,
    stringaProposta varchar(1000),
    stato int DEFAULT 0,
    dataProposta DATE DEFAULT CURRENT_DATE,
    oraProposta TIME DEFAULT CURRENT_TIME,
    dataValutazione DATE,
    oraValutazione TIME,
    UtenteP int NOT NULL,
    AutoreV int NOT NULL,
    StringaInserita varchar(1000),
    numerazione int,
    idPagina int,
    CONSTRAINT pk_modifica PRIMARY KEY(idModifica),
    CONSTRAINT fk_stringaInserita FOREIGN KEY(StringaInserita, numerazione, idPagina) REFERENCES fraseCorrente(stringaInserita, numerazione, idPagina) DELETE ON CASCADE,
    CONSTRAINT fk_AutoreV FOREIGN KEY(AutoreV) REFERENCES UTENTE(idUtente),
    CONSTRAINT fk_UtenteP FOREIGN KEY(utenteP) REFERENCES UTENTE(idUtente)
)
```
**- COLLEGAMENTO**
```SQL
CREATE TABLE COLLEGAMENTO
(
	stringaInserita varchar(1000),
	numerazione INTEGER,
	idPagina pk_utente,
	paginaCollegata pk_utente,
	PRIMARY KEY(StringaInserita, numerazione, idPagina, paginaCollegata),
	CONSTRAINT pk_fraseInserita FOREIGN KEY(StringaInserita, numerazione, idPagina) REFERENCES fraseCorrente(stringaInserita, numerazione, idPagina)
)
```
***TRIGGER***
**- quando un utente scrive per la prima volta una pagina, il suo ruolo passa da _utente_ ad _autore_**
```SQL
CREATE OR REPLACE FUNCTION diventaAutore() RETURNS TRIGGER AS $$
DECLARE
    ruoloUtente Utente.ruolo%TYPE;
BEGIN
   SELECT ruolo INTO ruoloUtente
   FROM UTENTE 
   WHERE idUtente = new.idAutore;

   IF ruoloUtente != 'Autore' THEN
           UPDATE Utente SET ruolo = 'Autore' WHERE idutente = new.idautore;
        RETURN new;
    ELSE
        RETURN NULL;
   END IF;
END;
$$ LANGUAGE plpgsql; 

CREATE OR REPLACE TRIGGER diventaAutore 
AFTER INSERT ON pagina
FOR EACH ROW
    EXECUTE FUNCTION diventaAutore();
```
**- quando una proposta di modifica viene fatto dall'autore della pagina, la proposta viene direttamente accettata**
```SQL
CREATE OR REPLACE FUNCTION ModificaAutore ()
RETURNS TRIGGER
AS $$
BEGIN
    IF new.utentep = new.autorev THEN
        UPDATE modificaproposta SET stato = 1 WHERE idmodifica = new.idmodifica;
        RETURN new;
    ELSE
        RETURN NULL;
    END IF ;
END;
$$
language plpgsql;

CREATE OR REPLACE TRIGGER ModificaAutore
AFTER INSERT ON modificaproposta
FOR EACH ROW
EXECUTE FUNCTION ModificaAutore()
```
**- quando viene inserita una nuova frase all'interno di una pagina, la numerazione viene impostata dal trigger**
```SQL
CREATE OR REPLACE FUNCTION inserimento_frase() RETURNS TRIGGER
AS $$
DECLARE
valore INTEGER = -1;
BEGIN
    SELECT numerazione INTO valore
    FROM frasecorrente
    WHERE new.idpagina = idpagina
    ORDER BY numerazione DESC LIMIT 1;
    IF valore != -1 THEN 
           new.numerazione = valore+1;
    ELSE
        new.numerazione = 0;
    END IF;
    RETURN new;
END;
$$
language plpgsql;

CREATE OR REPLACE TRIGGER inserimento_frase
BEFORE INSERT ON frasecorrente
FOR EACH ROW
EXECUTE FUNCTION inserimento_frase()
```
**- setta la data di valutazione e l'ora di valutazione quando viene accettata o rifiutata una proposta di modifica**
```SQL
CREATE OR REPLACE settaggioDataOraValutazione() RETURNS TRIGGER
AS $$
BEGIN
	UPDATE modificaProposta SET dataValutazione = NOW() WHERE idModifica = old.idModifica;
	UPDATE modificaProposta SET oraValutazione = NOW() WHERE idModifica = old.idModifica;
	RETURN new;
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE TRIGGER settaggioDataOraValutazione
AFTER UPDATE modificaProposta
FOR EACH ROW
WHEN (old.stato = 0)
	EXECUTE FUNCTION settaggioDataOraValutazione();
 ```

***PROCEDURE***

**- inserito un titolo, il nomeUtente, di chi deve creare la pagina, e il testo della pagina, la procedura crea la tupla nella tabella pagina e crea tuple per ogni frase. Il testo viene diviso in frasi all'interno della procedure.**
```SQL
CREATE OR REPLACE PROCEDURE crea_pagina(titoloScelto IN TEXT, nomeUtente IN TEXT, testoPagina IN TEXT)
AS $$
DECLARE
	frase varchar(100000) := '';
	pos_prec INTEGER := 1;
	pagina pagina.idPagina%TYPE;
	autore utente.idUtente%TYPE;
	pos INTEGER := 1;
	num INTEGER := 0;
	H RECORD;
	J RECORD;
    	k RECORD;
BEGIN
	SELECT idUtente INTO autore FROM utente WHERE LOWER(login) = LOWER(nomeUtente) LIMIT 1;
	
	INSERT INTO PAGINA(titolo, idAutore) VALUES (titoloScelto, autore);
	SELECT idPagina INTO pagina FROM PAGINA WHERE PAGINA.titolo = titoloScelto AND PAGINA.idAutore = autore AND PAGINA.dataOraCreazione = now();

	LOOP
		EXIT WHEN pos >= LENGTH(testoPagina);
		IF SUBSTRING(testoPagina FROM pos FOR 1) IN ('.', ',', ';', '!', '?') THEN
            frase := REGEXP_REPLACE(SUBSTRING(testoPagina FROM pos_prec FOR (pos - pos_prec)+1), '^\s+', '');
			pos_prec := pos + 1;
			CALL inserimento_frase(frase, pagina);
			-- INSERT INTO FRASECORRENTE(stringaInserita, numerazione, idPagina) VALUES (frase, num, pagina);
			RAISE NOTICE '%', frase;
		END IF;
		pos := pos + 1;
	END LOOP;

	IF pos_prec <= LENGTH(testoPagina) THEN
        frase := REGEXP_REPLACE(SUBSTRING(testoPagina FROM pos_prec FOR LENGTH(testoPagina)), '^\s+', '');
		frase := frase || '.'; 
		CALL inserimento_frase(frase, pagina);
		-- INSERT INTO FRASECORRENTE(stringaInserita, numerazione, idPagina) VALUES (frase, num, pagina);
		RAISE NOTICE '%', frase;
	END IF;
END;
$$
LANGUAGE PLPGSQL;
```
**- data una frase e l'idPagina, inserisce la frase all'interno del testo**
```SQL
CREATE OR REPLACE PROCEDURE inserimento_frase(stringa IN varchar(100), pagina IN pagina.idPagina%TYPE)
AS $$
DECLARE
valore INTEGER;
BEGIN
	SELECT numerazione INTO valore
	FROM frasecorrente
	WHERE pagina = idpagina
	ORDER BY numerazione DESC LIMIT 1;
	INSERT INTO frasecorrente (stringainserita,numerazione,idpagina) VALUES (stringa,valore+1,pagina);
END;
```
**- data una frase la procedura ricerca tutti i titoli che contengono la frase inserita al loro interno e mostra tutti i titoli e i testi.**
```SQL
CREATE OR REPLACE PROCEDURE ricerca_testi(titoloInserito IN varchar(100))
AS $$
DECLARE
	frase varchar(100000) := '';
	titolotesto varchar(100);
	stringa varchar(1000);
	pagina pagina.idpagina%type;
    num INTEGER;
    dataVal DATE;
    sProposta varchar(1000);
	H RECORD;
	J RECORD;
    k RECORD;
BEGIN
	FOR H IN (SELECT idPagina, titolo FROM PAGINA WHERE LOWER(titolo) LIKE '%' || LOWER(titoloInserito) || '%' ORDER BY titolo) LOOP
	RAISE INFO '%', 'TITOLO PAGINA:';
	RAISE INFO '%', H.titolo;
	
	CREATE TABLE tempVersione
    (
        stringaIns varchar(1000),
        numerazione INTEGER
    );
	

		FOR k IN (SELECT stringaInserita, numerazione, datainserimento FROM frasecorrente WHERE H.idpagina = idpagina ORDER BY numerazione ASC) LOOP
			SELECT numerazione, dataValutazione, stringaProposta INTO num, dataVal, sProposta
			FROM modificaproposta 
			WHERE H.idpagina = idpagina AND k.stringainserita = stringainserita AND numerazione = k.numerazione AND stato = 1 AND datavalutazione <= K.datainserimento
			ORDER BY dataValutazione DESC, oraValutazione DESC LIMIT 1;
			IF num IS NULL THEN 
				INSERT INTO tempVersione VALUES (k.StringaInserita, k.numerazione);
			ELSE
				INSERT INTO tempVersione VALUES (sProposta, num);
			END IF;
		END LOOP;
		FOR J IN (SELECT stringaIns FROM tempVersione) LOOP
			stringa := J.stringaIns;
			frase := frase || ' ' || stringa;
		END LOOP;
		RAISE INFO '%', frase;
		RAISE INFO '%', '';
		frase := '';
		DROP TABLE tempVersione;
	END LOOP;
	
END;
$$
LANGUAGE PLPGSQL;
```
**- data una data inserita si visualizza la versione piú recente fino a quella data**
```SQL
CREATE OR REPLACE PROCEDURE ricostruzione_versione(pagina IN pagina.idpagina%type, dataInsertita IN DATE)
AS $$
DECLARE
    frase varchar(1000);
    num INTEGER;
    dataVal DATE;
    sProposta varchar(1000);
    k RECORD;
BEGIN
    CREATE TABLE tempVersione
    (
        stringa varchar(1000),
        numerazione INTEGER
    );

    FOR k IN (SELECT stringaInserita, numerazione, datainserimento FROM frasecorrente WHERE pagina = idpagina) LOOP
        SELECT numerazione, dataValutazione, stringaProposta INTO num, dataVal, sProposta
        FROM modificaproposta 
        WHERE pagina = idpagina AND k.stringainserita = stringainserita AND numerazione = k.numerazione AND stato = 1 AND datavalutazione <= dataInsertita
        ORDER BY dataValutazione DESC, oraValutazione DESC LIMIT 1;
        IF num IS NULL THEN 
            INSERT INTO tempVersione VALUES (k.StringaInserita, k.numerazione);
        ELSE
            INSERT INTO tempVersione VALUES (sProposta, num);
        END IF;
    END LOOP;

    DROP TABLE tempVersione;
END;
$$
LANGUAGE PLPGSQL;
```
**- indicato un utente, viene mostrato il numero di notifiche da visionare e tutte le notifiche (titolo testo, frase da modificare e frase proposta) in ordine cronologico**
```SQL
CREATE OR REPLACE PROCEDURE visiona_notifiche(autore IN utente.idUtente%TYPE)
AS $$
DECLARE
    notifiche INTEGER := 0;
    fraseProposta varchar(100):= '';
    K RECORD;
BEGIN
    notifiche = numero_notifiche(autore);

    RAISE INFO '%', 'Ci sono '  notifiche  ' notifiche da visualizare';
    RAISE INFO '%', '';
    IF notifiche > 0 THEN 
        FOR K IN (SELECT * FROM modificaProposta WHERE idModifica IN (SELECT idModifica FROM notifica WHERE stato = 0 AND idAutore = autore) ORDER BY DataProposta ASC, oraProposta ASC)
        LOOP
            SELECT stringaProposta INTO fraseProposta FROM modificaProposta WHERE autorev = autore AND idPagina = K.idPagina AND numerazione = K.numerazione AND stato = 1 AND datavalutazione <= K.dataProposta AND oravalutazione < K.oraProposta ORDER BY datavalutazione DESC, oravalutazione DESC LIMIT 1;

            RAISE INFO '%', '---------------------------------------------------------------------------------------------------------------------------------';
            RAISE INFO '%', 'FRASE: '  fraseproposta;
            IF fraseProposta != '' THEN
                RAISE INFO '%', 'FRASE SELEZIONATA: '  fraseproposta;
            ELSE
                RAISE INFO '%', 'FRASE SELEZIONATA: '  K.stringaInserita;
            END IF;

            RAISE INFO '%', 'FRASE PROPOSTA: '  K.stringaProposta;
            RAISE INFO '%', '';
        END LOOP;
        RAISE INFO '%', '---------------------------------------------------------------------------------------------------------------------------------';
    END IF;
END;
$$
LANGUAGE PLPGSQL;
```
**- Indicato il nome utente di chi propone, l'id della pagina da modificare, la frase da modificare, la frase da proporre e la numerazione della frase, la procedura inserisce la modifica all'interno della tabella ModificaProposta**
```SQL
CREATE OR REPLACE PROCEDURE proponi_modifica(loginUtente IN Utente.login%TYPE, paginaSelezionata IN pagina.idpagina%TYPE, fraseDaModificare IN TEXT, fraseDaProporre IN TEXT, numFrase IN INTEGER )
AS $$
DECLARE
    autore Utente.idUtente%TYPE;
    utente Utente.idUtente%TYPE;
    fraseInserita TEXT := '';
BEGIN
    SELECT idUtente INTO utente FROM Utente WHERE login = loginUtente;
    SELECT idAutore INTO autore FROM Pagina WHERE idPagina = paginaSelezionata;

    SELECT stringaInserita INTO fraseInserita FROM FraseCorrente WHERE idpagina = paginaSelezionata AND fraseDaModificare = stringaInserita AND numerazione = numFrase LIMIT 1;
	
	IF fraseInserita IS NULL THEN
        SELECT stringaInserita INTO fraseInserita FROM ModificaProposta WHERE idpagina = paginaSelezionata AND fraseDaModificare = stringaProposta AND numerazione = numFrase LIMIT 1;
	END IF;

    INSERT INTO ModificaProposta(stringaProposta, utenteP, autoreV, stringaInserita, numerazione, idPagina)
    VALUES (fraseDaProporre, utente, autore, fraseInserita, numFrase, paginaSelezionata);

    RAISE INFO '%', 'Operazione completata con successo';
END;
$$
LANGUAGE PLPGSQL;
```
**- dato un titolo, la procedura mostra il testo della pagina**
```SQL
CREATE OR REPLACE PROCEDURE ricostruzione_testo(titoloInserito IN varchar(100))
DECLARE
	frase varchar(100000) := '';
	titolotesto varchar(100);
	stringa varchar(1000);
	pagina pagina.idpagina%type;
    num INTEGER;
    dataVal DATE;
    sProposta varchar(1000);
    k RECORD;
BEGIN
CREATE TABLE tempVersione
    (
        stringaIns varchar(1000),
        numerazione INTEGER
    );
	
	SELECT idPagina, titolo INTO pagina, titolotesto FROM PAGINA WHERE LOWER(titolo) LIKE '%' || LOWER(titoloInserito) || '%' ORDER BY titolo DESC LIMIT 1;
	RAISE INFO '%', titolotesto;

    FOR k IN (SELECT stringaInserita, numerazione, datainserimento FROM frasecorrente WHERE pagina = idpagina ORDER BY numerazione ASC) LOOP
        SELECT numerazione, dataValutazione, stringaProposta INTO num, dataVal, sProposta
        FROM modificaproposta 
        WHERE pagina = idpagina AND k.stringainserita = stringainserita AND numerazione = k.numerazione AND stato = 1 AND datavalutazione <= K.datainserimento
        ORDER BY dataValutazione DESC, oraValutazione DESC LIMIT 1;
        IF num IS NULL THEN 
            INSERT INTO tempVersione VALUES (k.StringaInserita, k.numerazione);
        ELSE
            INSERT INTO tempVersione VALUES (sProposta, num);
        END IF;
    END LOOP;
	
	FOR K IN (SELECT stringaIns FROM tempVersione) LOOP
	stringa := K.stringaIns;
	frase := frase || ' ' || stringa;
	END LOOP;
	
	RAISE INFO '%', frase;
    DROP TABLE tempVersione;
END;
$$
LANGUAGE PLPGSQL;
```
**- dato un id di una pagina, restituisce le frasi della pagina con la posizione**
```SQL
CREATE OR REPLACE PROCEDURE numerazione_frasi(paginaSelezionata IN pagina.idpagina%type)
AS $$
DECLARE
    frase varchar(1000);
    num INTEGER;
    dataVal DATE;
	dataCreazione DATE;
    sProposta varchar(1000);
    k RECORD;
BEGIN
    CREATE TABLE tempVersione
    (
        stringa TEXT,
        numerazione INTEGER
    );	
    FOR k IN (SELECT stringaInserita, numerazione, datainserimento FROM frasecorrente WHERE paginaSelezionata = idpagina) LOOP
		SELECT numerazione, dataValutazione, stringaProposta INTO num, dataVal, sProposta
        FROM modificaproposta 
        WHERE paginaSelezionata = idpagina AND k.stringainserita = stringainserita AND numerazione = k.numerazione AND stato = 1
        ORDER BY dataValutazione DESC, oraValutazione DESC LIMIT 1;
        IF num IS NULL THEN 
            INSERT INTO tempVersione VALUES (k.StringaInserita, k.numerazione);
        ELSE
            INSERT INTO tempVersione VALUES (sProposta, num);
        END IF;
    END LOOP;
	
 FOR k IN (SELECT * FROM tempVersione ORDER BY numerazione ASC) LOOP
			RAISE INFO '%', K.numerazione || ' - ' || k.stringa;
 END LOOP;
   
    DROP TABLE tempVersione;
END;
$$
LANGUAGE PLPGSQL;
```

***FUNZIONI***

**- Restituisce il numero di notifiche che l'autore deve ancora visualizzare**
```SQL
CREATE OR REPLACE FUNCTION numero_notifiche(autore IN utente.idUtente%TYPE) RETURNS INTEGER
AS $$
DECLARE
    notifiche INTEGER := 0;
BEGIN
    SELECT COUNT(*) INTO notifiche
    FROM NOTIFICA N JOIN MODIFICAPROPOSTA MP ON N.idModifica = MP.idModifica
    WHERE N.stato = 0 AND N.idAutore = autore;

    RETURN notifiche;
END;
$$
LANGUAGE PLPGSQL;
```
