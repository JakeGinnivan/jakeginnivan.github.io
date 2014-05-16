---
layout: post
title: "BDDfy v4.0 Examples Support"
date: 2014-05-05 16:09:00 +0000
comments: true
---

One of the major features in BDDfy v4 is the inclusion of examples

**Scenario:** Successful rail card purchases  
**Given:** the buyer is a *&lt;buyer category&gt;*  
**And:** the buyer selects a *&lt;fare&gt;*  
**When:** the buyer pays  
**Then:** a sale occurs with an amount of *&lt;price&gt;*  

Examples:

    | Buyer Category | Fare          | Price |
    | Student        | Monthly Pass  | $76   |
    | Senior         | Monthly Pass  | $98   |
    | Standard       | Monthly Pass  | $146  |
    | Student        | Weekly Pass   | $23   |
    | Senior         | Weekly Pass   | $30   |
    | Standard       | Weekly Pass   | $44   |
    | Student        | Day Pass      | $4    |
    | Senior         | Day Pass      | $5    |
    | Standard       | Day Pass      | $7    |
    | Student        | Single Ticket | $1.5  |
    | Senior         | Single Ticket | $2    |
    | Standard       | Single Ticket | $3    |

This is an example of a Cucumber test with examples, for each row in the examples table the test will run effectively giving us a data driven BDD style test. 
<!-- more -->
I have picked this test because it is a bit more complex than the standard `Given I have 3 beers in my fridge, When I drink 2, Then I have 1 beer left`.. Yay, how interesting..

In this example, I will use BDDfy's fluent API to write this test. First without examples:

    this.Given(_ => TheBuyerIsA(BuyerCategory.Student))
        .And(_ => TheBuyerSelectsA(new MonthlyPass()))
        .When(_ => TheBuyerPays())
        .Then(_ => ASaleOccursWithAnAmountOf(new Money(76)))
        .BDDfy("Successful rail card purchases");

Which gives us a nicely formatted output of

    Scenario: Successful rail card purchases
      The buyer is a Student
        The buyer selects a Monthly Pass
      The buyer pays
      A sale occurs with an amount of $76.00

The next step is to turn this into a test with examples.

### Step 1
Create somewhere to put the examples, BDDfy supports private/public fields and properties as well as local variables in your test method. The following all work:

    private Fare fare;
    private BuyerCategory _buyerCategory;
    public Money Price { get; set; }
    
    public void MyTest()
    {
          var Fare = default(Fare);
    }

Local variables are the cleanest and recommended approach, but feel free to use fields or properties when examples are shared between tests.

### Step 2
Then we should replace the arguments we are passing to our steps to be the fields/properties

    var buyerCategory = default(BuyerCategory);
    var fare = default(Fare);
    var price = default(Money);
    
    this.Given(_ => TheBuyerIsA(buyerCategory))
        .And(_ => TheBuyerSelectsA(fare))
        .When(_ => TheBuyerPays())
        .Then(_ => ASaleOccursWithAnAmountOf(Price))
        .BDDfy("Successful rail card purchases");

### Step 3
Use the .WithExamples() extension method to pass your examples to BDDfy

    this.Given(_ => TheBuyerIsA(buyerCategory))
        .And(_ => TheBuyerSelectsA(fare))
        .When(_ => TheBuyerPays())
        .Then(_ => ASaleOccursWithAnAmountOf(price))
        .WithExamples(new ExampleTable(
            "Buyer Category", "Fare", "Price")
        {
            { BuyerCategory.Student, new MonthlyPass(), new Money(76) },
            { BuyerCategory.Senior, new MonthlyPass(), new Money(98) },
            { BuyerCategory.Standard, new MonthlyPass(), new Money(146) },
            { BuyerCategory.Student, new WeeklyPass(), new Money(23) },
            { BuyerCategory.Senior, new WeeklyPass(), new Money(30) },
            { BuyerCategory.Standard, new WeeklyPass(), new Money(44) },
            { BuyerCategory.Student, new DayPass(), new Money(4) },
            { BuyerCategory.Senior, new DayPass(), new Money(5) },
            { BuyerCategory.Standard, new DayPass(), new Money(7) },
            { BuyerCategory.Student, new SingleTicket(), new Money(1.5m) },
            { BuyerCategory.Senior, new SingleTicket(), new Money(2m) },
            { BuyerCategory.Standard, new SingleTicket(), new Money(3m) }
        })
        .BDDfy("Successful rail card purchases");

BDDfy supports two ways of providing examples, if you are migrating from SpecFlow and you are using simple types (unlike the example above where my examples are actually classes) then we support the normal text table:

    .WithExamples(@"
        | Header 1 | Header 2     | Header3    |
        | Value 1  | 2            | 3          |
        |          | 14 Mar 2010  | Transition |");

### Step 4
Thats it, simply run your tests and the output will be something like this:

    Scenario: Successful rail card purchases
      The buyer is a <Buyer Category>        
        The buyer selects a <Fare>           
      The buyer pays                         
      A sale occurs with an amount of <Price>

    Examples: 
    | Buyer Category | Fare         | Price   |
    | Student        | Monthly Pass | $76.00  |
    | Senior         | Monthly Pass | $98.00  |
    | Standard       | Monthly Pass | $146.00 |
    | Student        | Weekly Pass  | $23.00  |
    | Senior         | Weekly Pass  | $30.00  |
    | Standard       | Weekly Pass  | $44.00  |
    | Student        | Day Pass     | $4.00   |
    | Senior         | Day Pass     | $5.00   |
    | Standard       | Day Pass     | $7.00   |
    | Student        | Day Pass     | $1.50   |
    | Senior         | Day Pass     | $2.00   |
    | Standard       | Day Pass     | $3.00   |

Errors will be reported against examples rather than steps when examples are used. For example:
    
    Examples: 
    | Buyer Category | Fare         | Price   | Error                                                  |
    | Student        | Monthly Pass | $76.00  |                                                        |
    | Senior         | Monthly Pass | $98.00  |                                                        |
    | Standard       | Monthly Pass | $146.00 |                                                        |
    | Student        | Weekly Pass  | $23.00  | ChuckedAWobblyException: $24 should be $23 [Details 1] |
    | Senior         | Weekly Pass  | $30.00  |                                                        |
    | Standard       | Weekly Pass  | $44.00  |                                                        |
    | Student        | Day Pass     | $4.00   |                                                        |
    | Senior         | Day Pass     | $5.00   |                                                        |
    | Standard       | Day Pass     | $7.00   |                                                        |
    | Student        | Day Pass     | $1.50   |                                                        |
    | Senior         | Day Pass     | $2.00   |                                                        |
    | Standard       | Day Pass     | $3.00   |                                                        |

    Exceptions:
    1. ChuckedAWobblyException: $24 should be $23
    <stacktrace>

This works with all of our built in reports. If you have custom reports then you will get a test for *each* example. Have a look at the built in reporters for examples on how to build custom reporters with examples support.

## Other things to note

### Step names
There are multiple ways to put placeholders in step names, the one shown above allows you to simply use the parameter on the step method. If it matches a column in your example table it will be substituted in the step name. If it doesn't match you will just get the value like you currently do.

We also support the following:

    GivenIHave__initialCount__Beers() # Then access the value from a field/property directly in the step
    .Given(_ => InitialCount(), "Given I have <initial count> beers")

If you have more ideas on how we could make this even better, let us know!

### Errors
BDDfy will try it's best to tell you when something is wrong, for example your test will fail if examples in your example table are not put anywhere. This is handy when you make a spelling mistake or refactor things and they no longer match.
We will also give you row and column information of values if they cannot be assigned, for example trying to assign 'null' to a value type.


I really hope you enjoy using this feature, we have wanted to add this to BDDfy for a long time but it is a pretty massive feature so it has taken us a while. We also not only wanted to make it as good as the Gherkin first languages, but take advantage of the fact that BDDfy tests are written in code to allow complex types to be used in examples.
