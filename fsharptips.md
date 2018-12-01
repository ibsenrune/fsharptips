# I gang med F&#35;

intro

## Model binding

Efter en af mine "F# - why, how, and for what?" præsentationer blev jeg spurgt, hvorfor model binding ikke virkede med F# record typer.

En `Address` type som

```fsharp
type Address = {
    Street : string
    City : string
}
```

bliver af compileren oversat til IL, som svarer til følgende C# kode:

```csharp
[CompilationMapping(SourceConstructFlags.RecordType)]
[Serializable]
public sealed class Address :
    IEquatable<Address>,
    IStructuralEquatable,
    IComparable<Address>,
    IComparable,
    IStructuralComparable
{
    [DebuggerBrowsable(DebuggerBrowsableState.Never)]
    internal string Street@;

    [DebuggerBrowsable(DebuggerBrowsableState.Never)]
    internal string City@;

    [CompilationMapping(SourceConstructFlags.Field, 1)]
    public string City
    {
        get
        {
            return this.City@;
        }
    }

    [CompilationMapping(SourceConstructFlags.Field, 0)]
    public string Street
    {
        get
        {
            return this.Street@;
        }
    }

    public Address(string street, string city)
    {
        this.Street@ = street;
        this.City@ = city;
    }
}
```

Her kan vi se grunden til, at ASP.NET ikke kan udføre model binding imod denne type: de to properties, `City` og `Street`, har ingen `public` `set`ter.

Heldigvis er det nemt at løse: ved at bruge attributten [`CLIMutable`](https://msdn.microsoft.com/en-us/visualfsharpdocs/conceptual/core.climutableattribute-class-%5Bfsharp%5D) kan vi fortælle compileren, at vi ønsker record typen oversat til en IL type med en default constructor og `public` `get`tere og `set`tere:

```fsharp
[<CLIMutable>]
type Address = {
    Street : string
    City : string
}
```

Ser vi på, hvad denne type compiles til, finder vi

```csharp
[CLIMutable]
[CompilationMapping(SourceConstructFlags.RecordType)]
[Serializable]
public sealed class Address : IEquatable<Address>, IStructuralEquatable, IComparable<Address>, IComparable, IStructuralComparable
{
    [DebuggerBrowsable(DebuggerBrowsableState.Never)]
    internal string Street@;

    [DebuggerBrowsable(DebuggerBrowsableState.Never)]
    internal string City@;

    [CompilationMapping(SourceConstructFlags.Field, 1)]
    public string City
    {
        get
        {
            return this.City@;
        }
        set
        {
            this.City@ = value;
        }
    }

    [CompilationMapping(SourceConstructFlags.Field, 0)]
    public string Street
    {
        get
        {
            return this.Street@;
        }
        set
        {
            this.Street@ = value;
        }
    }

    public Address(string street, string city)
    {
        this.Street@ = street;
        this.City@ = city;
    }

    public Address()
    {
    }
}
```

Her ser vi, at compileren har dannet en default constructor og at `City` og `Street` har `public` `set`ters. 

Dermed kan ASP.NET nu instantiere et objekt af typen `Address` og initialisere hver property. Herefter kan ASP.NET model binde til `Address` typen.

Med andre ord: at få model binding i ASP.NET til at virke med F# record typer kræver blot, at typerne annoteres med [`CLIMutable`](https://msdn.microsoft.com/en-us/visualfsharpdocs/conceptual/core.climutableattribute-class-%5Bfsharp%5D) attributten.