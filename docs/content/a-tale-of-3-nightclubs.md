# A Tale of 3 Nightclubs

This C# tutorial is based on a [Scalaz tutorial](https://gist.github.com/oxbowlakes/970717) by Chris Marshall and was originally ported to [fsharpx](https://github.com/fsprojects/fsharpx/blob/master/tests/FSharpx.CSharpTests/ValidationExample.cs) by Mauricio Scheffer.

Additional resources:

* Railway Oriented Programming by Scott Wlaschin - A functional approach to error handling
	* [Blog post](http://fsharpforfunandprofit.com/posts/recipe-part2/)
    * [Slide deck](http://www.slideshare.net/ScottWlaschin/railway-oriented-programming)
    * [Video](https://vimeo.com/97344498)

## Part Zero : 10:15 Saturday Night

We start by referencing Chessie and opening the ErrorHandling module and define a simple domain for nightclubs:

    [lang=csharp]
    using Chessie.ErrorHandling;
    using Chessie.ErrorHandling.CSharp;


    enum Sobriety { Sober, Tipsy, Drunk, Paralytic, Unconscious }
    enum Gender { Male, Female }

    class Person
    {
        public Gender Gender { get; private set; }
        public int Age { get; private set; }
        public List<string> Clothes { get; private set; }
        public Sobriety Sobriety { get; private set; }

        public Person(Gender gender, int age, List<string> clothes, Sobriety sobriety)
        {
            this.Gender = gender;
            this.Age = age;
            this.Clothes = clothes;
            this.Sobriety = sobriety;
        }
    }

Now we define some validation methods that all nightclubs will perform:

    [lang=csharp]
    class Club
    {
        public static Result<Person, string> CheckAge(Person p)
        {
            if (p.Age < 18)
                return Result<Person, string>.FailWith("Too young!");
            if (p.Age > 40)
                return Result<Person, string>.FailWith("Too old!");
            return Result<Person, string>.Succeed(p);
        }

        public static Result<Person, string> CheckClothes(Person p)
        {
            if (p.Gender == Gender.Male && !p.Clothes.Contains("Tie"))
                return Result<Person, string>.FailWith("Smarten up!");
            if (p.Gender == Gender.Female && p.Clothes.Contains("Trainers"))
                return Result<Person, string>.FailWith("Wear high heels!");
            return Result<Person, string>.Succeed(p);
        }

        public static Result<Person, string> CheckSobriety(Person p)
        {
            if (new[] { Sobriety.Drunk, Sobriety.Paralytic, Sobriety.Unconscious }.Contains(p.Sobriety))
                return Result<Person, string>.FailWith("Sober up!");
            return Result<Person, string>.Succeed(p);
        }
    }

## Part One : Clubbed to Death

Now let's compose some validation checks via syntactic sugar and LINQ:

    [lang=csharp]
    class ClubbedToDeath
    {
        public static Result<decimal, string> CostToEnter(Person p)
        {
            return from a in Club.CheckAge(p)
                   from b in Club.CheckClothes(a)
                   from c in Club.CheckSobriety(b)
                   select c.Gender == Gender.Female ? 0m : 5m;
        }
    }

Let's see how the validation works in action:

    [lang=csharp]
    var Dave = new Person(Gender.Male, 41, new List<string> { "Tie", "Jeans" }, Sobriety.Sober);
    var costDave = ClubbedToDeath.CostToEnter(Dave); // Error "Too old!"

    var Ken = new Person(Gender.Male, 28, new List<string> { "Tie", "Shirt" }, Sobriety.Tipsy);
    var costKen = ClubbedToDeath.CostToEnter(Ken);  // Success 5


We can even use pattern matching on the result:

    [lang=csharp]
    var Ruby = new Person(Gender.Female, 25, new List<string> { "High heels" }, Sobriety.Tipsy);
    var costRuby = ClubbedToDeath.CostToEnter(Ruby);
    costRuby.Match(
        (x, msgs) => {
            Console.WriteLine("Cost for Ruby: {0}", x);
        },
        msgs => {
            Console.WriteLine("Ruby is not allowed to enter: {0}", msgs);
        });

The thing to note here is how the Validations can be composed together in a computation expression.
The type system is making sure that failures flow through your computation in a safe manner.

## Part Two : Club Tropicana

Part One showed monadic composition, which from the perspective of Validation is *fail-fast*. That is, any failed check short-circuits subsequent checks. This nicely models nightclubs in the real world, as anyone who has dashed home for a pair of smart shoes and returned, only to be told that your tie does not pass muster, will attest.

But what about an ideal nightclub? One that tells you *everything* that is wrong with you.

Applicative functors to the rescue!

Let's compose some validation checks that accumulate failures using LINQ sugar:

    [lang=csharp]
    class ClubTropicana
    {
        public static Result<decimal, string> CostToEnter(Person p)
        {
            return from c in Club.CheckAge(p)
                   join x in Club.CheckClothes(p) on 1 equals 1
                   join y in Club.CheckSobriety(p) on 1 equals 1
                   select c.Gender == Gender.Female ? 0m : 7.5m;
        }
    }

The usage is the same as above except that as a result we will get either a success or a list of accumulated error messages from all the checks. 

Dave tried the second nightclub after a few more drinks in the pub:

    [lang=csharp]
    var daveParalytic = new Person(
        age: 41,
        clothes: new List<string> { "Tie", "Shirt" },
        gender: Gender.Male,
        sobriety: Sobriety.Paralytic);

    var costDaveParalytic = ClubTropicana.CostToEnter(daveParalytic);

    costDaveParalytic.Match(
        ifSuccess: (x, msgs) => Console.WriteLine("Cost for Dave: {0}", x),
        ifFailure: errs => Console.WriteLine("Dave is not allowed to enter:\n{0}", String.Join("\n", errs)));

This is the result:

    Dave is not allowed to enter:
    Too old!
    Sober up!

So, what have we done? Well, with a *tiny change* (and no changes to the individual checks themselves), we have completely changed the behaviour to accumulate all errors, rather than halting at the first sign of trouble. Imagine trying to do this using exceptions, with ten checks.

## Part Three : Gay bar

And for those wondering how to do this with a *very long list* of checks here is a solution:

    [lang=csharp]
    class GayBar
    {
        public static Result<Person, string> CheckGender (Person p)
        {
            if (p.Gender == Gender.Male)
                return Result<Person, string>.Succeed(p);
            return Result<Person, string>.FailWith("Men only");
        }

        public static Result<decimal, string> CostToEnter(Person p)
        {
            return new List<Func<Person, Result<Person, string>>> { CheckGender, Club.CheckAge, Club.CheckClothes, Club.CheckSobriety }
                .Select(check => check(p))
                .Collect()
                .Select(x => x[0].Age + 1.5m);
        }
    }


The usage is the same as above:

    [lang=csharp]
    var person = new Person(
        gender: Gender.Male,
        age: 59,
        clothes: new List<string> { "Jeans" },
        sobriety: Sobriety.Paralytic);
    var cost = GayBar.CostToEnter(person);
    cost.Match(
        ifSuccess: (x, msgs) => Console.WriteLine("Cost for person: {0}", x),
        ifFailure: errs => Console.WriteLine("Person is not allowed to enter:\n{0}", String.Join("\n", errs)));
