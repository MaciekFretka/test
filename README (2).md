# Raport z laratorium 4

# Do zrobienia

1.  Zalogować się w programie 'keygenme', korzystając z numeru indeksu, jako login, i hasło, które trzeba znaleźć na podstawie skompilowanego programu.

2.  Napisać skrypt pozwalający generować hasło dla programu 'keygenme' dla łatwego logowania.

# Przebieg pracy

## 1. Znalezienie klucza

Po otworzeniu programu 'keygenme' wprowadzamy swój numer indeksu i jakieś hasło. Z prawdopodobieństwem bliskim do 100% nie da się zalogować z powodu nieznanego hasła.

Skoro nie mamy kodu źródłowego programu, sprobujemy przeczytać instrukcje procesorowe za pomocą dwóch narzędzi `objdumb` i `gdb`.

Komenda `objdump -D ./keygenme` pozwala przeczytać wszystkie instrukcje programu w języku assembler (intel). Skoro język assembler nie jest bardzo czytelny, najprostszym podejściem jest znalezienie `main` tego programu, z którego właśnie i zaczyna się praca aplikacji. W czasie wyszukiwania main'a można zauważyć, że program składa się z kilku funkcji, które nazwane w czytelny sposób, ale w tym momencie po prostu zwracamy ta to uwagę. Patrząc na main myróżniam taki mechanizm działania programu:

1. alokacja jakiejś pamięci (caloc@plt)
2. alokacja jakiejś pamięci (caloc@plt)
3. alokacja jakiejś pamięci (caloc@plt)
4. wypisać dane (puts@plt)
5. wypisać dane (puts@plt)
6. wypisać dane (puts@plt)
7. wypisać dane (puts@plt)
8. wyczyścić pamięć (fflush@plt)
9. wypisać dane (\_\_printf_chk@plt)
10. wczytać dane (\_\_isoc99_scanf@plt)
11. dalej wczytujemy dane (getchar@plt)
12. wypisać dane (\_\_printf_chk@plt)
13. dalej wczytujemy dane (fgets@plt)
14. jakaś własna funkja (generate_key), po nazwie można zrobić założenie, że to generacja klucza
15. porównanie 2-ch stringów (strncmp@plt)
16. coś podobnego na go to (main+0xe1)
17. jakaś własna funkja (winner), po nazwie można zrobić założenie, że coś, co będzie jeśli wygrywasz
18. coś podobnego na go to (main+0xe6)
19. jakaś własna funkja (winner), po nazwie można zrobić założenie, że coś, co będzie jeśli przegrywasz
20. oczyszczenie pamięci (free@plt)
21. oczyszczenie pamięci (free@plt)
22. zamknięcie programu (nop nop ...)

Jeśli patrzeć na odpowiedniki kodu improwizowany mechanizm programu i na komunikację z użytkownikiem w samym programie można zauważyć, że krok 10,11 to wczytanie loginu, a 13 - hasła.

Następnym krokiem odtwarzam program za pomocą debagera gdb (`$ gdb ./keygenme`). Za pomocą komendy `$ disassemble main` wypisuję zawartość programu, wyszukuję linijkę zawierającą `strncmp@plt` i stawię na niej breakpoint (`$ b *(main+206)`). Tę linijkę wybrałem, ponieważ to jest porównywanie czegoś z czymś (3 paramenty: string1, sting2, rozmiar string'a). Uruchamiam program w debagierze (`$ run`), wpisuję login, hasło. za tym debager wstrzymuję program.

Wiadomo, że `strncmp@plt` przyjmuję 3 argumenty: pierwszy string, drugi string, ich rozmiar. Przed wywołaniem `strncmp@plt` widzimy 3 razy push na stos ('esp' - wskaźnik na stos): `push 0x20` (to liczba, sprawdzamy `$ p/d 0x20` = 32), `push %esi`(to sting, sprawdzamy `$ x/s $esi` = '5ny61djxvfpo73qgi980ktzl2rsue4hm', akkurat 32 znaki!) i `push %ebx` (to sting, sprawdzamy `$ x/s $ebx` = 'test\n', to co wprowadziłem jako hsło).

W wyniku na stosie mamy takie rzeczy (komenda `$ x/4xw $esp`) : `0x0804c1a0 0x0804c1d0 0x00000020 0x080484a4`. Patrzymy w te obszary pamięci: `$ x/s 0x0804c1a0` = "test\n", `$ x/s 0x0804c1d0` = "5ny61djxvfpo73qgi980ktzl2rsue4hm", `$ p/d 0x00000020` = 32

![gdb1.png](gdb1.png)

Robię następny wynik: mam funkcję, która porównuje 2 string'a. Mam w debagerze przed wywołaniem porównywania przemieszczenie na stos 2-ch stringów i 1-j liczby. Jednym stringiem jest wprowadzony przeze mnie string, długość innego jest równa liczbie. Być może ta 32 znakowa liczba i jest hasłem?

Sprawdzamy w aplikacji `$ ./keygenme`, wprowadzam nr indeksu: 245816 i wprowadzam potencjalnie hasło ('5ny61djxvfpo73qgi980ktzl2rsue4hm'). Wszystko się udało!

![good.png](good.png)

## 2. Napisanie skrypta

Przyjmując to, co zostało zrobione podczas pierwszego zadania, załóżmy, że funkcja `generate_key` generuję klucz.

Uruchomiamy debager, ustalamy jako breakpoint pierwszą linijkę funkcji `generate_key`, uruchomiamy i wprowadzamy dane do testowania (`$ gdb ./keygenme`, `$ b *(generate_key+1)`, `$run` ).

Jak program się wstrzymał możemy debagować. Od razu powiem, że moje wiedzy assemlera nieduże i nie pewne, nawet kiedy wiem jak musi działać komenda, lepiej sprawdzić przez debagera: czy prawidłowo pomyślałem.

Lista komend do gdb, z których korzystałem ( rejestrami są eax,ebx,edx,esp,esi,esp i inne; obszary pamięci zapis typu 0x0000000):

- `x/s $rejestr` - pokazuje zawartość rejestru jako string (najczęściej mamy zapis typu /123/534/, który nie jes czytelnym, lub błąd dostępu do pamięci. wtedy używamy z p/d ...)
- `x/s $obszar_pamięci` - pokazuje zawartość obszaru pamięci jako string (czasem mamy zapis typu /123/534/, który nie jes czytelnym, lub błąd dostępu do pamięci. wtedy używamy z p/d ...)
- `p/d $rejestr` - pokazuje liczbowo zawartość rejestru
- `p/d $obszar_pamięci` - pokazuje liczbowo zawartość obszaru pamięci
- `x/4xw $stosie` - pokazuje ostatnie 4 zapisy na stosie (przykład stosów: esp, esi, edi)
- `$ b *(generate_key+n)` - ustala breakpoint z przesunięciem odnośnie początku funkcji 'generate_key' o 'n'
- `disassemle` - wyświetla bieżącą funkcję i linijkę, na której się program został wstrzymany
- `n 1` - przejść do następnego breakpoint'a

Breakpointy ustaliłem, prawię we szystkich, linijkach funkcji. Przejdziemy do otrzymanych przez mnie wartości (login wpisany '245816', hasło 'test')

Jak czytać zapis: `nr_linijki [wartości_przed_przejściem_linijki]=>wartości_po_przejściu (może być jeszcze jakaś dodatkowa informacja, co się dzieje w jakiejś komendzie):

```
0

1 [...] => ecx=36

. tu kilka linijek, które nie są ciekawe ponieważ po prostu zachowanie na stos

.

16 [ebp= -12568, 134514564, 245816, 134529488]=> ebx=245816

19 [...] => esi=134529488

22 [ebx=245816 | eax=134529440("test\n") | ecx=36|edx=0] => eax=245816 | ebx=245816

24 [esi=""|ebx=245816 | eax=245816 | ecx=36 | edx=0] => eax DIV ecx = eax + edx => eax=6828 | ebx=245816 | ecx=36 | edx=8

26 [...] => cl=100 => ecx=100

28 [eax=6828 | ebx=245816 | ecx=36 | edx=8] => eax=245816 | ebx=245816 | edx=8 | ecx=100

30 [edx=8 | ecx=100 | eax=245816 | ebx=245816 | edi=2022700/-134722020] => edi=8

32 [edx=8] => edx=0

34 [ecx=100 | eax=245816 | ebx=245816 | edx=0 ] => eax=2458 | ebx=245816 | ecx=100 | edx=16

36 [cl=100]=>cl=0

38 [eax=2458 | ebx=245816 | ecx=0 | edx=16] => eax DIV ecx = eax + edx => eax=2458 | ebx=2458 | ecx=100 | ebx=245816

40 [ebx=2458] => ebx=0

43 [ebp=""(-12568),134514564,245816,""(134529488)...] => czytamy co mamy w ebp i robimy zdjęcie ;)

46 goto 87

87 [ebp(0x00000020=32) |ecx=0 | zf=0,cf=0] => zf=0 | cf=0

90 [0x10(ebx)=32 | ecx=0] goto 48, jeśli ecx < 0x10(ebx) => goto 48

48 [ecx=0 | edi=8] => ecx+(edi\*1)=8 => edx=8

51 [ebx=0] => ebx=36

56 [edx=8 | eax=2458] => eax=8

58 [edx=8] => edx=0

60 [eax=8 | ebx=36 | ecx=0 | edx=0 | cl=0] => eax DIV ebx = eax + edx => eax=0|ebx=36|ecx=0|edx=8

62 [zf=0 | cf=0 | cl=0 | ecx=0] => cmpl 0 != -0x14(%ebp) => zf=0|cf=0|cl=0

66 if (0 != -0x14(%ebp)) goto 77 else 68

68 [eax=0] => eax=35

73 [eax=35 | ebx=36 | ecx=0 | edx=8] => eax SUB edx (eax-edx) => eax=27 | ebx=36 | ecx=0 | edx=8

75 [eax=27 | ebx=36 | ecx=0 | edx=8] => edx=27

77 [al=27 | 0x8048890="usr2lztk089igq37opfvxjd16yn5wacbmh4e" | edx=27] => al=53 | 0x8048890="usr2lztk089igq37opfvxjd16yn5wacbmh4e" | edx=27|eax=53

83 [al=53 | ecx=0] => ecx=0 | al=53 | esi=0x00000035=53

86 [ecx=0] => ecx=1

87 [ecx=1 | 0x10(ebp)=245816 lub-134230068 | cf=0,zf=0] => zf=0|cf=0

90 [0x10(ebx)=32 | ecx=1] goto 48, jeśli ecx < 0x10(ebx) => goto 48

48 [ecx=1 | edi=8 | edx=27] => edx=9

51 [ebx=36] => ebx=36

56 [edx=9 | eax=53|]
```

Jak widać jeszcze z przesunięcia 87 i 90, mamy cykl od 0 do 32 (akurat tyle długość hasła). W numerze 77 mamy string'a, który ma 36 znaków, liczba 53 to kod znaku '5' (z niego zaczyna się hasło). tzn z tego string'a bierzemy litery i ustalamy na stos esi. Jeśli porównać ten string z hasłem otrzymanym w zadaniu 1, to jest to samo, ale w innym porządku i z pewnego miejsca.

Patrząc na linijkę 77 widać skąd się bierze numer litery (al=27). Dalej trzeba w odwrotnym porządku prześledzić, w jaki sposób otrzymaliśmy tą liczbę.

Dalej zacząłem pisać wzór na otrzymanie tej litery 27 patrząc w odwrotnym porządku na liczby w zapisi wyżej:

```
27 = edx = eax - edx = 35 - edx = 35 - ( eax % ebx ) = 35 - ( eax % 36) = 35 - ( edx % 36) = 35 - ( (ecx+(edi*1)) % 36 ) = 35 - ( (ecx+(edi*1)) % 36) = 35 - ( (ecx+(edx*1)) % 36) = 35 - ( (ecx+(eax%ecx*1)) % 36) = =  35 - ( (ecx+((245816 % 36)*1)) % 36)
```

Specjalnie nie rozwijałem dalej ecx, bo on zależy od iteracji cyklu. Cykl zaczyna się z ecx=0 i idzie, dopóki ecx!=32. Skoro każda iteracja cykla zwiększa ecx o 1 (linijka 86), mamy 32 iteracji.

Z powyższego wynika wzór `nr_litery = 35 - ( ( nr_iteracji + (login % 36)\*1 ) % 36)`

Funkcja dla znalezienia hasła w języku go (cały program w `main.go`):

```
func encoder(login int) string {
	key := "usr2lztk089igq37opfvxjd16yn5wacbmh4e"
	position := (len(key) - 1) - (login % len(key))
	var password string
	for len(password) <= 32 {
		if position < 0 {
			position = len(key) - 1
		}
		password += string(key[position])
		position--
	}
	return password
}
```

## Przykład pracy:

![finish.png](finish.png)

Można uruchomić skompilowaną wersję `./lab`

# Wnioski
