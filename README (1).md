# Raport z Laboratorium 4 (exploitme)

# Cel:

# Wprowadzenie

Stos (ang. Stack) – liniowa struktura danych, w której dane dokładane są na wierzch stosu i z wierzchołka stosu są pobierane (bufor typu LIFO, Last In, First Out; ostatni na wejściu, pierwszy na wyjściu)(Wikipedia).

Jak to działa w asemblerze intel: Mamy obszar pamięci, który zwykle przedstawiamy jako kolumnę. Kiedy dodajemy do niego dane (push), dane są umieszczone na górę i wskaźnik wskazujący na stos wskazuje na początek tych danych, mówimy, że stos rośnie. Przy tym wcześniej umieszczone dane znajdują się pod nowymi i przemieszczają się odnośnie wskaźnika na stos. Przy odebraniu danych ze stosu (pop) pobieramy dane z góry i wskaźnik na stos teraz wskazuje na bardziej stare dane, które pod spodem.

![stack.png](stack.png)
![push_pop.png](push_pop.png)

# Przebieg pracy

Dla tworzenia aplikacji należało wprowadzić jako parametr jakiś 21 znakowy klucz. Ale po wprowadzeniu 21 znaków i więcej nic się nie zmieniło, ale zostało zauważono, że po wprowadzeniu 51 znaków mamy błąd "segmentation fault". Otrzymujemy ten błąd wtedy, gdy miejsce przeznaczone dla pewnych danych jest za małe i nadpisuje dane, które znajdują się pod nimi na stosie, a przy próbie odwołania do starych danych, one już wskazują na inne obszary pamięci, do których możemy nie mieć dostępu.

Pierwszym zadaniem było wypisać do konsoli tekst(BAM! /bin/bash), który będzie po wprowadzeniu prawidłowego parametru.

Otworzyłem program za pomocą `objdump -D ./exploitme`. Następnym krokiem znalazłem funkcję main i przeanalizowałem jakie inne funkcje wywoływane z niego. Funkcja `validate` przyciągnęła moją uwagę. Patrząc na tą funkcję widać, że niezależnie od wartości do rejestru ecx idze 0, a później mamy taką umowę że w dowolnym przypadku wskakujemy do funkcji `looser`, która wyświetla, że wprowadziliśmy niepoprawne dane. Skoro nie możemy wpływać na to porównanie, analizujemy wcześniejszy kod. Skoro przy dużej ilości danych mamy segmentation fault być może gdzieś możemy z tego skorzystać. W funkcji `validate`
mamy nie nie bezpieczną funkcję do kopiowania string'ów (strcpy). Ta funkcja kopiuję dane nie sprawdzając ilość tych danych, co pozwala zapisywać dane na stosie. Po tej funkcji mamy operację `ret` która pobiera z góry stosu wskaźnik do przejścia na pewną linijkę kodu. To znaczy przy odpowiednim nadpisaniu stosu możemy zmienić miejsce z którego będzie iść program po operacji `ret`. Skoro blisko mamy niebezpieczną funkcję która nadpisuję stos i do tego ta funkcja pracuję z parametrem wprowadzone przez użytkownika, możemy z tego skorzystać: wprowadzić takie dane, aby nadpisać dane na stosie i przejść do potrzebnej linki.

Analizując inne funkcje programu, widać, że funkcja `destroy_world` wypisuję akurat potrzebne dane. To znaczy że nadpisujemy dane na stosie w taki sposób, żeby przejść z `ret` do tej funkcji. Funkcja zaczyna się w komórkę pamięci nr 0x080485d7. Metodą Trial and Error że amy nadpisywanie stosu przy więcej niż 50 znaków (nie licząc \n w końcu każdej linijki). W wyniku musimy jako parametr programu wprowadzić 50 dowolnych znaków i nr komórki pamięci. to łatwo zrobić za pomocą perl'a bo on pozwala przekształcać liczby z systemu szesnastkowego do string. Otrzymujemy taką komendę:

```
./exploitme  `perl -e 'print "1"x51 . "\xd7\x85\x04\x08"'`

```

W drugim zadaniu mieliśmy zostawić nadpis "BAM! /bin/bash", ale usunąć segmentation fault, spowodowany '\n' na końcu linijki. Dla tego musimy w taki sposób nadpisać stos, żeby po skończeniu funkcji `destroy_world` wywołać normalne zamknięcie programu (`exit()`). Analizując main programu na samym końcu mamy wywołanie tej funkcji. Adres tej części kodu
0x080486fc. Teraz już wiadomą drogą dopisujemy parametr do wywołania programu tak, żeby nadpisać w pewnym miejscu stos wartością 0x080486fc. Z tego powodu, że `ret` pracuję jako pop do stosu, i w przypadku funkcji `destroy_world` nie zmienia się wskaźnik dla powrotu z funkcji, możemy po prostu popisać nasz parametr. Otrzymujemy:

```
./exploitme  `perl -e 'print "1"x51 . "\xd7\x85\x04\x08" . "\xfc\x86\x04\x08"'`

```

Trzecim zadaniem było zwrócić przy wyjściu z programu pewny błąd. Wiedząc, że `exit()` przyjmuję 1 parametr, który znajduję się na górze stosu musimy po prostu zastąpimy jeszcze kawałek stosu o dowolną liczbę. Otrzymujemy:

```
./exploitme  `perl -e 'print "1"x51 . "\xd7\x85\x04\x08" . "\xfc\x86\x04\x08" . "\x10"'
```

Czwartym zadaniem było wywołać bash z poziomu programu. Tak się wydarzyło, że mamy w programie wywoływanie funkcji systemowej `system()`, która wywołuje to c trafi jej do parametru. Ale w jaki sposób wywołać bash? W poprzednich zadaniach do konsoli zostało wypisano "BAM! /bi/bash". `/bin/bash` to ścieżka do wywoływania programu bash. Pozostało tylko wprowadzić ten string do `system()`. Dla tego zwrócimy uwagę na funkję która wypisuję tekst do konsoli `destroy_world`. Przed wydrukowaniem (`call 80484b4 <puts@plt>`) wrzucamy tą linijkę na stos (`mov %eax,(%esp)`). Za pomocą gdb wstrzymujemy program w tej linijce (0x080485f1) i sprawdzamy zawartość pamięci której adres przechowywany w eax(to "/bin/bash"). To znaczy że jeśli umieścić ten wskaźnik na obszar pamięci od `system()` odpala się bash. Mając doświadczenie z zadań 1, 2 i 3, wprowadzimy taki parametr do programu, że zastąpimy na stosie 2 zapisy wymieniając wartości na 0x080486f0 (wywołanie `system()`) i 0x80487d0 (zawiera "/bin/bash"). Otrzymujemy:

```
./exploitme  `perl -e 'print "1"x51 . "\xf0\x86\x04\x08" . "\xd0\x87\x04\x08" . "\xfc\x86\x04\x08" . "\x10"'`
```

## Wyniki:

![wyniki.png](wyniki.png)

# Wnioski

Aby zapobiec łamań programu przez przepełnienie stosu należy korzystać z takich funkcji lub budować program w taki sposób, żeby wszystkie wartości z którymi współdziałał użytkownik były zawsze sprawdzane na rozmiar.
