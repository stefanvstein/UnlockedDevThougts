---
title: "Fibo"
date: 2025-03-27
tags: ["test", "layout", "stil"]
---

I en studiecirkel om DDD fälldes en rolig kommentar, att DDD inte passar FP för att, något i stil med: för att man i FP använder ett språk där man implementerar fibonacci serien med rekursion. Det kan ha varit någon annan beräkning som ofta används som exempel för att introducera någon i FP. Jag kommer inte ihåg.

Nåväl, här är ett exempel på just fibo räknande i Clojure:

```clojure
(defn fib []
  ((fn fibo [a b]
       (cons a (lazy-seq (fibo b (+ a b)))))
    0 1))
```

Jag räknar inte med att alla ni i studiecirkeln förstår hälften av vad jag pratar om här, men för er som är intresserade.

I det här exemplet är vi ganska rakt på sak, och implementerar fibo sekvensen, på en ganska låg nivå. Visst är det ganska kort, elegant, om än kryptiskt för den som inte kan just Clojure. Vi visar tydligt att vi skapar serien som en lazy sekvens, och det är väl viktigt i det här fallet, eftersom fibo går mot oändligheten och vi vet ju inte hur många tal anroparen behöver. Att den är lazy betyder att sekvensen realiseras sent efter behov.

Nedan tar vi 50 tal ur sekvensen genom att anropa fib och ta 50 tal ur sekvensen den returnerar. Vi räknar därmed inte ut mer än 50 tal ur serien.

```clojure
(take 50 (fib))
```

Följande fib funktion tycker jag är snyggare rent estetiskt. Den är fortfarande sent realiserad, men använder funktioner som map och iterate, som de flesta programmerare, som stött på ett funktionellt språk, känner igen.

```clojure
(defn fib []
  (let [fibo (fn [[a b]]
               [b (+ a b)])]
    (map first (iterate fibo [0 1]))))
```

Den gör i princip samma sak, och vi kan ta 50 sent realiserade tal ur serien på samma sätt:

```clojure
(take 50 (fib))
```

Språket Haskell är lite kul på just fibo serien, och används ofta som exempel för att tänka lite djupare. Haskell som språk, är lazy, så argument till en funktion realiseras inte före anropet, utan inne i funktionen, när de behövs, om de nu någonsin behövs. Denna naiva Haskell fibo implementation är ren, och ser precis ut som den mattematiska definitionen av fibo serien:

```
fib 0 = 0
fib 1 = 1
fib n = fib (n-1) + fib (n-2)
```

Och vi kan ta 50 första talen, genom t.ex:

```
take 50 (map fib [0..])
```

Naturligtvis är denna lazy, eftersom Haskell är lazy i sig, och den är mer generell då vi anger att vi tar 50 från serien som börjar med index 0 och framåt. Stiligt. Men, denna fibo funktion har en brist. Den räknar ut serien om, och om igen för varje tal. Helt ok mattematiskt, men vi använder en dator. Så det går fort från början, med allt eftersom, går det saktare och saktare. Processorfläkten börjar surra. Nedan funktion löser detta, men är inte lika estetisk, om man nu inte är hängiven Haskellist.

```
fib n = fibs !! n
  where
    fibs = 0 : 1 : zipWith (+) fibs (tail fibs)

take 50 (map fib [0..])
```

I C# är det också enkelt att implementera fibo, på ett lazy vis. T.ex:

```java
IEnumerable<ulong> fibo()
{
  ulong a = 0, b = 1, c;
  yield return a;
  yield return b;
          
  while (true)
  {
    c = a + b;
    a = b;
    b = c;
    yield return c;
  }
}
```

Det är inte funktionellt, och det smäller efter ett tag, då C# inte automagiskt uppgraderar primitiver till oändligt breda tal, som de flesta funktionella språken brukar göra.

Men vi kan ta de 50 första talen, på liknande vis som i de funktionella språken:

```
Enumerable.Take(fibo(), 50);
```

Du kan säkert göra den här C# koden bra mycket bättre. Jag har inte så många års erfarenhet av C#.

Sen tar vi Java. Det är väl enkelt att räkna fibo serien i Java. Men skall vi göra det lazy, med JDK som ända bibliotek, så blir det ganska mycket kod, då vi inte har något stöd för lazyness:

```java
class Fib implements Iterator<Long>, Iterable<Long> {
  long a = 0, b = 1, c = 0;
  int count = 0;

  @Override
  public Long next() {
    count++;
    if (count == 1)
      return a;
    else if (count == 2)
      return b;
    else {
      c = a + b;
      a = b;
      b = c;
    }
    return c;
  }

  @Override
  public boolean hasNext() {
    return true;
  }

  @Override
  public void remove() {
    throw new UnsupportedOperationException();
  }

  @Override
  public Iterator<Long> iterator() {
    return new Fib();
  }
```

Något som liknar funktionen take, har vi inte heller, så ”here we go”:

```java
static <T> Iterable<T> take(final int num, final Iterable<T> iterable) {
    return new Iterable<T>() {

      @Override
      public Iterator<T> iterator() {
        return new Iterator<T>() {
          int i = 0;
          Iterator<T> it = iterable.iterator();

          @Override
          public boolean hasNext() {
            return it.hasNext() && i < num;
          }

          @Override
          public T next() {
            if (hasNext()) {
              i++;
              return it.next();
            }
            throw new NoSuchElementException();
          }

          @Override
          public void remove() {
            it.remove();
          }
        };
      }
    };
  }
```

At last:
```
take(50, new Fib())
```
Inget fel på Java, men JDK’t saknar de generella komponerbara funktionerna, som jag tror jag påpekade saknades både 1 och 2 gånger i min Java 7 föreläsning. Förhoppningsvis kommer de med Closures i Java 8.

Visst, det finns en hel del 3:e partare som har en hel del generella funktioner. De finns bara inte i det annars så feta JDK’t.

Nog om rekursion.

Vi pratade också igår om att herr Evans fanns med i en lite nyare intervju, som kan vara kul att lyssna på:

http://www.infoq.com/interviews/Technology-Influences-DDD

Vad säger han här om FP och DDD:

*What about functional programming. How do you see DDD mapping to this new world? 
Yes, well, again, I think that what you need in order to do DDD is a powerful abstraction tool, something that allows you to take that ubiquitous language and render it in code. So, in functional, we certainly have the abstraction power and a lot of people in functional programming tend to think in terms of basically creating their own internal DSLs, that is Domain Specific Languages. So that’s one style of programming which people in functional tend to use and say, well, build up a language basically that you can use to describe your domain. So that’s probably one of the areas I expect we’ll work out. We’ll end up with more DSL-like style. And I think this kind of internal DSLs was one of the popular styles within the OO world although it never spread so widely, but it was influential.*

*I think another is that when you start looking at problem, for example, I’ve been looking at functional programming for the last few years and I’ve done a little bit of work in Clojure, for example and partly because I really feel, I mean, you know, I want to, I get excited about new stuff too, and so I wanted to get my hands on, you know, something functional. I chose Clojure because it’s a really pure functional language and you can’t cop out and do it the OO way in Clojure, so it forces you to really think about it as a functional problem.
And one thing I noticed, and in retrospect this doesn’t seem surprising to me at all, is that the nature of the models and the way that I approach the problems starts to change. And, one of the things I noticed is that I’m much more likely to view things as sort of streams and sequences. So you look at your domain and you start looking for the things where you’ve got a stream of things happening or stream of things you can process. Because functional language is sort of structured that way and you’ve got a function and you fit it as stream and out comes another stream of something.*

*I guess my point is, that is has the abstraction power that we need. There is, if you take the mentality of how can I express the ubiquitous language here and what shape should my model take? If you’re really open-minded about those things you’ll find ways of creating nice DDD solutions with functional and then we’ll need to do patterns and variations of the old patterns. Some of them are natural like value objects which was always one of the things I used to preach about a lot. But, in OO you have to kind of say, you know, make an object like this. In functional you almost start there. That’s the default way that a data structure, at least, would be. It is not an object and you still have to say, “Well these functions and this data structure together accomplish this thing.” And in Clojure, the concept of a protocol is pretty clear. So anyway, value objects map over pretty easily, very naturally.*

*Then entity, what is an entity? The identity part of an entity is still relevant, but some of the other aspects of it may just not apply. Domain events? I think that’s going to be one of the real keys to making DDD work in functional is to think more in terms basically of event sourcing which was one of the, you know, has been one of the big trends in DDD for the last several years. And now, when I start looking at problems in the functional world I say, “Ah, event sourcing actually fits pretty well.” A sequence of events again, that stream, being processed to create basically a current view of an entity. So the entity isn’t there as such, it’s not stored state, but if you want to know what a particular entity looks like at a particular point of time, you take the stream of events you put it together, I mean that’s event sourcing. Works pretty well with functional, actually. So we’re just at the beginning of this, you know, that’s the exciting thing, it’s fun. I mean, well, we’re not really at the beginning I’m sure because functional is an old, old style of programming, but we’re at the beginning of maybe of widespread adoption.*

Kul att han gillar trevliga FP språk :-)

I Clojure-presentationen, jag höll tidigare, landade vi efter en massa funktionella idéer, och lisps ovanliga syntax, tillslut vid Protocol. Protocol är en defnition av fria funktioner, som kan användas för polymorphism på typ. Dvs, samma typ av polymorphism som vi är vana vid i OO. Annars kan vi ju använda polymorphism på vad som behagas i ett funktionellt språk, då det är funktionellt. Men, eftersom Clojure är designad att köra på host:ade plattformar, så är polymorphism på typ ett bra val, både ur prestanda och interrop perspektiv.

Ett Protocol påminner om interface, en rent abstract data typ, men ingår inte i någon typ hierarki. Den är fri.
Ett exempel på Protocol, för ett bank konto, skulle kunna vara:

```clojure
(defprotocol account-protocol
  (deposit [this amount])
  (withdraw [this amount]))
```
…som definierar två fria funktioner deposit och withdraw.

I ett Protocol tar varje funktion en instans av en typ som första argument. Här kan olika typer av typer användas, beroende av vad de skall användas till. Typen Type är generell, kraftfull, och kan t.om innehålla, den funktionella-guden förbjude, variabler. Reifieringar är anonyma, och väldigt koncisa, medan Records är speciellt utformade för att vara just domäntyper.

 ```clojure
 (defrecord account [balance transactions])
 ```

…definierar domän-datatypen account, som innehåller en balans och transaktioner, som förmodligen innehåller ett tal som motsvarar balans och någon sekvens  som motsvarar de transaktioner som kontot ingått i.

I Clojure programmerar vi mycket med associativa typer, t.ex. hash mappar, och där finns många funktioner som gör det koncist. Ofta ersätts klasser i OO, av mappar med namn och värden och/eller funktioner. Tänker man efter så är en klass, i ett dynamiskt språk, helt enkelt inte mycket mer än en associativ samling med namn och data, rent logiskt. Objekt hierarkier blir helt enkelt nästlade mappar.

En definition:

```clojure
(def olle {:name "Olle",
           :last-name "Andersson",
           :details {:age 23, :favorite-dish "Pizza"},
           :adress {:street "Strandvägen", :num 43}}
```

Där olle är ett värde, en associativ samling med nycklarna :name, :last-name, :details, och :adress, där :details pekar på ytterligare en associativ, med nyklarna :age och :favorite-dish.

Med funktionsanropet:  
`
(assoc-in olle [:details :favorite-dish] "Paneng gai")
`  
…skapar vi ett nytt värde där som olle, troligen varit på semester i thailand.

Medan  anropet:  
`(update-in a [:details :age] inc)`  
…motsvarar en olle som fyllt år.

En Record är en typ som ser ut som en mapp, men som realiseras som en vanlig klass i host plattformen. Vi har därmed en map, med prestanda som, och ev. typsäkerhet som, motsvarar en java klass. Vi har fortfarande en en massa kraftfulla standardfunktioner, generellt designade att arbeta med associativa samlingar, eller sekvenser av data i största allmänhet, som fungerar på en record lika väl.

Om:  
`
(def a (account. 23.4 []))  
`  
…definierar a som ett konto med balansen 23.4 och dess transaktioner som en tom vector, så justerar
(assoc a :balance 45.2)
a genom att returnera ett nytt värde där nyckeln :balance har värdet 45,2.


Vi använde funktionen (assoc [map key value]), en generell funktion för att associera nycklar med värden i associativa samlingar, för att skapa ett nytt värde av typen account. Datatypen account, är fortfarande en domän typ, som också råkar vara en vanlig java klass i en jvm. Visst, kanske inte så tillåtet för ett konto, men vi skapade ett nytt värde.

Anropet:  
`
(assoc {} :foo "Bar")
`  
motsvarar tom, assoierad med :foo och "Bar", som då motsvarar den enkla hashmappen: {:foo "Bar"}

Domäntyper, Records, kan naturligtvis vara nästlade, precis som vanliga mappar, och till o med utökas, precis som en vanlig map, med nya nycklar, om så behövs, då den är en associativ. En Record är oförändringsbar, precis som andra associativa i Clojure, så, vi får hela tiden nya account object, och därmed inga överraskningar.


Åter till Protocol: Om vi vill att vår Record account, skall ingå i protokollet account-protocol, som definierade de fria funktionerna deposit och withdraw, skulle den förslagsvis definieras som:

```clojure
(defrecord account [balance trans]
  account-protocol
  (deposit [this value]
    (assoc this :balance (+ balance value)
                :trans (conj trans value)))
  (withdraw [this value]
    (assoc this :balance (- balance value)
                :trans (conj trans (- value)))))
```

..och våra implementationen för våra fria deposit och withdraw funktioner, när de tar ett account som första argument är definierade att skapa nya account, med justerad balans och sekvens av transaktioner. Vi kan fortfarande behandla account som en map, men, vi kan även anropa våra fria funktioner för att utföra mer korrekt konto funktionalitet. Kom ihåg att vi alltid skapar nya värden, så våra map eller data sekvens funktioner, gör aldrig våld.


Funktionen (conj col x), betyder conjoin, och lägger till något, x, till en samling, col, med mest effektiva metod.

Anropet:
```clojure
(deposit (account. 32 []) 43)
..ger en ny account:
{:balance 75 :transactions [43]}
```

Vi kan också låta java klasser, t.ex. String ingå i account-protocol, även om det i ett sånt här exempel kanske är svårt att förstå vad det betyder semantiskt. Men:

```clojure
(extend-type java.lang.String
  account-protocol
   (deposit [_ v] (str v))
   (withdraw [_ v] (str v)))
```

Och deposit på en sträng "Java":  
`
(deposit "Java" "Clojure")
`  
returnerar en annan sträng "Clojure".

Konstigt!? Vi lät java.lang.String vara ett konto.  Men, den här akrobatiken, att förbättra en annan typhierarki, som vi kanske inte ens har kontroll över, är något som är väldigt svårt i de flesta OO språk, som i det här fallet Java.  Antingen får vi där göra våld på den främmande hierarkin, eller så får vi använda oss utav adapters/bryggor och därmed förlora identitet, och därefter kontinuerligt hamna i identitetskriser.  Det här är ett känt problem, som brukar kallas för ”The expression problem”. Att det i många OO språk är svårt kombinera två existerande hierarkier.  Här vänder vi på steken, och löser problemet med funktion. Klassen String är inte förändrad, vi löskopplar dispatch på typ.


Elegansen som sådana problem löses i FP, gör att: vad som domänmodelleras i OO med Coca cola och Marlboro  Lights, domänmodelleras i FP med Cognac och en fet cigarr. Modelleringen blir inte bunden av OO’s avigsidor.


Enkel hantering av oförändringsbarhet, är en annan storhet som gör modellering enklare i FP. Kanske inte lättare, men enklare.


F#, Haskell och andra ML språk är lika, om inte mer, smidiga på domänmodellering som Clojure. Men i dessa, har vi statiskt typsystem som kan vara svårt att hantera innan man är van, och kan tyckas vara lite väl hårda ibland, tycker jag. 
