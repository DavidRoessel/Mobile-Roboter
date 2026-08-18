# Mobile-Roboter
KIT Projekt zum Bauen und programmieren eines Roboters. Der Roboter soll am Ende des Semesters in der Lage sein einen Hindernisparkour zu überwinden. Der Roboter nutzt den Mikrocontroller STM-32 Nucleo-L432KC. Als Entwicklungsumgebung wurde das Programm STM32CubeIDE 2.1.1 der Firma STMicroelectronics verwendet. Die Programmierung ist in C.

Softwaredokumentation
1. Architektur und Design
Die Steuerungssoftware des Roboters (main.c) basiert auf einer kooperativen Multitasking-Architektur. Um eine flüssige Ausführung aller Subsysteme zu garantieren, wird vollständig auf blockierende Funktionen wie HAL_Delay() verzichtet.
Die Hauptschleife while(1) ruft kontinuierlich alle Tasks auf. Jede Task besitzt einen eigenen lokalen Timer basierend auf HAL_GetTick() und wird in einem festen Zeitraster (meist 20 ms, entsprechend 50 Hz) ausgeführt. Ist die Zeit noch nicht abgelaufen, gibt die Task die Kontrolle sofort mittels return; an die Hauptschleife zurück. Dies garantiert, dass zeitkritische Routinen wie die Odometrie verlustfrei im Hintergrund laufen.
2. Sensor-Datenverarbeitung
2.1 A/D-Wandlung (DMA)
Die analogen Sensoren (Liniensensoren und Rad-Encoder) werden über den ADC (Analog-Digital-Converter) ausgelesen. Um die CPU zu entlasten, nutzt der ADC den DMA (Direct Memory Access) Modus. Die Werte werden automatisch hardwareseitig in das Array buffer[6] geschrieben. Die Callback-Funktion HAL_ADC_ConvCpltCallback synchronisiert diese Daten regelmäßig in das Arbeits-Array adc[6], um korrupte Lesezugriffe während der Wandlung zu verhindern.
2.2 Odometrie (Schmitt-Trigger)
Die gefahrene Distanz wird durch optische Encoder an den Rädern gemessen. Da die analogen Signale bei wechselnden Lichtverhältnissen und hohen Geschwindigkeiten rauschen können, ist in taskOdometry ein Software-Schmitt-Trigger implementiert. Dieser nutzt unterschiedliche Schwellenwerte für den Wechsel von Weiß auf Schwarz (SCHMITT_HIGH) und von Schwarz auf Weiß (SCHMITT_LOW). Diese Hysterese verhindert Fehlzählungen bei unsauberen Signalen.
3. Aktorik (Motorsteuerung)
Die Ansteuerung der Motoren erfolgt über die Funktion motorControl(char dir, float speed). Sie abstrahiert die komplexe Timer-Konfiguration. Die Richtung: Festlegung der H-Brücken-Phasen (z.B. phase2_L_Pin).
Die Motorgeschwingkeit wird über die Funktion pwm() gesteuert. Sie rechnet Prozentwerte (0.0 bis 1.0) in 16-Bit-Timer-Ticks (0 - 65535) um. Invertierte Motor-Kanäle werden dabei automatisch mathematisch korrigiert (65535 * (1 - speed)). Dadurch können in anderen Funktionen die Motoren einfach gesteuert werden.

4. Der Haupt-Zustandsautomat (RaceState_t)
Der Roboter durchläuft den Wettbewerb anhand eines übergeordneten Zustandsautomaten (current_race_state), der in der main-Funktion verarbeitet wird. Dieser Automat hat 5 Zustände:
	STATE_PARCOURS: Start des Rennens, fährt den vordefinierten Streckenabschnitt ohne Linienführung rein über Odometrie ab.
	STATE_FOLLOW_LINE: Der Standard-Fahrmodus. Der Roboter zentriert sich mittels P-Regler auf der schwarzen Linie.
	STATE_SEARCH_LINE: Wird ausgelöst, wenn alle Sensoren "weiß" melden (Linie verloren). Ein Such-Algorithmus tastet die Umgebung systematisch ab.
	STATE_OVERCOME_GAP: Schlägt die Suche fehl, fährt der Roboter zeitgesteuert blind geradeaus, um eine Lücke zu überwinden.
	STATE_AVOID_OBSTACLE: Wird bei Auslösen der physischen Bumper-Taster aktiviert, um ein Hindernis sicher zu umfahren.
5. Beschreibung der Kern-Tasks
5.1 taskParcours (Odometrie-Navigation)
Dieser Modus steuert den Roboter über den gelben Start-Parcours. Distanzen und Kurvenwinkel sind in Arrays (path_distances, path_turns) vordefiniert. Die taskDriveStraight sorgt während der Geradeausfahrt mithilfe eines P-Reglers (error = encoder_ticks_left - encoder_ticks_right) für einen exakten Gleichlauf beider Räder, um Drift zu kompensieren.
5.2 taskLineFollower (Linienverfolgung)
Sobald die Liniensensoren Schwarz detektieren (Wert > LINE_THRESHOLD_BLACK), berechnet die Task den Fehler aus der Differenz des linken und rechten Sensors. Ein proportionaler Regelkreis (P-Regler) drosselt das kurveninnere Rad und beschleunigt das kurvenäußere Rad, um den Roboter kontinuierlich auf der Linie zu halten.
5.3 taskSearchLine (Suchalgorithmus)
Verliert der Roboter die Linie, beginnt er einen schrittweise iterierenden Suchvorgang (SEARCH_RIGHT → SEARCH_LEFT → SEARCH_CENTER → SEARCH_DRIVE_FWD). Die Task ist mit Sicherheits-Timeouts (safety_timeout = 1500 ms) ausgestattet, um bei blockierten Rädern nicht endlos im Suchzustand zu verharren.
5.4 taskAvoidObstacle (Trapez-Hindernisumfahrung)
Diese Task führt bei Kontakt mit einem Hindernis eine Trapez-Umfahrung durch:
	Kurzes Zurücksetzen (AVOID_BACK).
	Aufbau eines seitlichen Sicherheitsabstands (AVOID_FWD1).
	Vorbeifahrt parallel zur Rennlinie (AVOID_FWD_PARALLEL).
	Geometrisch bereinigtes Ausrichten (AVOID_ALIGN) vor der Rückgabe an den Line-Follower.
Dynamischer Failsafe: Sollte das Hindernis unerwartet lang sein und es bei der Rückkehr zur Linie zu einem erneuten Crash kommen, löst der Failsafe aus (AVOID_FAILSAFE_BACK etc.). Der Roboter bricht das Manöver ab, stellt sich wieder parallel und verlängert die Umfahrungsstrecke dynamisch, bis das Objekt sicher passiert wurde.

