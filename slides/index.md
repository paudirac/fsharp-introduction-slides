- title : F# introduction
- description : Introduction to F#
- author : Pau Cervera
- theme : night
- transition : default

***

- data-background : images/delorean.png

<h2 style="color:blue">F# introduction</h2>


<!-- <section> -->
<!--     <p class="fragment highlight-current-blue">Motivation</p> -->
<!--     <p class="fragment highlight-current-blue">Use case</p> -->
<!--     <p class="fragment highlight-current-blue">Scripting</p> -->
<!--     <p class="fragment highlight-current-blue">Theory</p> -->

<!-- <p class="fragment grow">grow</p> -->
<!-- <p class="fragment shrink">shrink</p> -->
<!-- <p class="fragment fade-out">fade-out</p> -->
<!-- <p class="fragment current-visible">visible only once</p> -->

<!-- <p class="fragment highlight-red">highlight-red</p> -->
<!-- <p class="fragment highlight-green">highlight-green</p> -->
<!-- <p class="fragment highlight-blue">highlight-blue</p> -->
<!-- </section> -->

***

### Motivation

***

### Typical C# day

    [lang=cs]
    public interface ILaboratoriesServiceContract
    {
        Laboratory GetLaboratoryById(long id);

        IList<Laboratory> FindLaboratories(LaboratoryFilter filter);

        long AddLaboratory(Laboratory laboratory);

        bool UpdateLaboratory(Laboratory laboratory);

        bool DeleteLaboratory(long IdLaboratory);
    }

---

#### actually, this interface has too many responsabilites (is not SOLID)

---

#### anyway, we can try to implement a *smart* solution

---

I said *smart*, because it was mine

---

# ANYWAY

---

### Wrap results that can fail

    [lang=cs]
    public interface ILaboratoriesServiceContract
    {
        SvcResult<Laboratory> GetLaboratoryById(long id);

        SvcResult<IList<Laboratory>> FindLaboratories(LaboratoryFilter filter);

        SvcResult<long> AddLaboratory(Laboratory laboratory);

        SvcResult<bool> UpdateLaboratory(Laboratory laboratory);

        SvcResult<bool> DeleteLaboratory(long IdLaboratory);
    }

***

    [lang=cs]
    public class SvcResult<T>
    {
        public T Value { get; private set; }

        public SvcResult(T value)
        {
            this.Value = value;
        }

        public SvcResult(T value, string error)
            : this(value)
        {
            if (!string.IsNullOrEmpty(error))
            {
                this.Errors = new List<string>();
                this.Errors.Add(error);
            }
        }

       // some code removed...
    }


***

    [lang=cs]
    public ActionResult Edit(long laboratoryId)
    {
        var res = this.svc_laboratory.GetById(laboratoryId);
        if (res.HasErrors)
        {
            string msg = string.Join(System.Environment.NewLine, res.Errors.ToArray());
            return new HttpNotFoundResult(msg);
        }
        else return View("Form", new LayoutableModelBase()
        {
            Id = laboratoryId,
            Model = res.Value,
        });
    }

---

### Seems not so **bad**

---

### but...

if you have to **compose** somehow the results of one service with another ones...

---

    [lang=cs]
        public SvcResult<bool> Assign(MassiveFamilies assignation)
        {
            MassiveFamiliesBusinessRules rules = new MassiveFamiliesBusinessRules(assignation, context);
            if (!rules.AreValid())
            {
                return new SvcResult<bool>(false, rules.GetErrors());
            }
            else
            {
                foreach (int familyId in assignation.Ids)
                {
                    //updating calibration by using selected status
                    var family = svc_families.GetById(familyId);
                    if (family.HasErrors || family.Value == null)
                        throw new ApplicationException("Impossible. There is no way to retrieve an unexisting family by id " + familyId);
                    family.Value.CustomerId = this.context.Current.CustomerId;
                    var res = svc_families.Create(family.Value);
                    if (res.HasErrors) //throws
                        throw new ApplicationException(string.Join(" ", res.Errors));
                }
                return new SvcResult<bool>(true);
            }
        }

***

![This is heavy, Doc](images/heavy-doc.gif)

> This is heavy, Doc!
(Marty McFly, two days ago)

***

## Enter F# magic

![Enter F# magic](images/doc.jpg)

***

- Expressiveness
- Type Safety
- Conciseness

***

### Expressiveness

    let updateParameters (t:int) (s:int) (tps:Protected<Api.TranslatedProtocol>) =

        checkHash "concurrency error, someone changed the same protocol while you were editing it"
        >=> checkIntegrity "data integrity error, there are parameters that are not from this protocol"
        >=> validateIdentifiers
        >=> checkTranslationKeyNotDefault DEFAULT_TRANSLATION_KEY (sprintf "%s is not a valid TranslationKey" DEFAULT_TRANSLATION_KEY)
        >=> checkTranslationKeysUniques "duplicated TranslationKey in this protocol"
        >=> prepareChanges
        >=> Utils.applyChanges "unable to apply changes"
        >> Utils.log "prot updated" "prot not updated:" <| (t, s, tprot)


***

### Expressiveness (2)

    (* Query parser *)
    let private pquery =
        pAll
        <|> pPlugin
        <|> pLeftAndRight
        <|> pKeyAndValue
        <|> pKey
        <|> pValue
        <|> pIdentifier
        <|> pName
        <|> pVersion
        <|> pNaive

    let private filterPlugins q config =
        let pres = match run pquery q with
                        | Success(result, a, b) -> Some result
                        | Failure(error, _, _) -> None


***

### Type Safety

    let printmsg = sprintf "%d bottles of beer, standing on a wall"
    in printmsg 42

***

### Conciseness

    module Application =

        let registerDependencies () =
            (* DI ? We don't need DI where we are going *)
            let path = ProtocolsEditor.Infrastructure.Config.protocols_file_path
            let protocols = ProtocolsEditor.Application.ReadProtocols.getProtocols path
            ProtocolsEditor.Application.Repositories.Protocols.initializeRepo protocols.Value |> ignore

            let translations = ProtocolsEditor.Application.Persistence.Translations.loadFromDirectoryOf path
            ProtocolsEditor.Application.Repositories.Translations.initializeRepo translations |> ignore

***

## How?

***

### Data types

- Algebraic data types
- Product types
- Immutable types

***


### The road to F#

Let's begin with some *inocuous* C# code

***

    [csharp]
    private bool Launch(Missiles m)
    {
        ThereIsNoPeopleArround() && m.Launch();
    }

---

##### (Mostly inocuous.)

---

##### At least the method is private!

---

anyway...

***

### How && works?

![&&](http://www.diracdelta.co.uk/science/source/t/r/truth%20table/image001.gif)

from **A** and **B** the computer can calulate **A&&B**

---

(no rocket science, at this point)

***

### but...

this is the same as saying that **A&&B** is

- **B** if **A** succeeds,
- and is false otherwise

***

### so

    [csharp]
    ThereIsNoPeopleArround() && m.Launch();

- if **&&** had not **short-circuited** the evaluation,
- independently of the value of ThereIsNoPeopleArround()

the missiles will have been launched

---

:(

***

## [ROP](http://fsharpforfunandprofit.com/posts/recipe-part2/) type

    type Result<'a,'b> =
        | Success of 'a
        | Failure of 'b

***
- data-background: images/doc-math.jpg

<h2 style="color:#00f">Interlude: function composition</h2>

***

$ f: A \mapsto B $

$ g: B \mapsto C $

then

$ h: A \mapsto C $

where

$ h(x) = (g \cdot f)(x) = g(f(x)) $

---

(I'm sorry I had to)

---

### Now in code

    let compose f g x = f (g x)

then when we have 2 particular f & g

    let h = compose f g
    let a = h 3

---


    let comose f g x = f (g x);;

    val comose : f:('a -> 'b) -> g:('c -> 'a) -> x:'c -> 'b

---

### We don't like to write **compose**

    let (>>) g f x = f (g x)

or **point free style**

    let (>>) g f = compose f g

then when we have 2 particular f & g

    let h = g >> f
    let a = h 3

---

### We don't like to lose the meaning of what we're doing

the idiomatic F# uses pipes instead

    let a = 3 |> g |> f

---

### Pipe

    x |> f = f x

---

### Reverse pipe

    f <| x = f x

---

# WAT?!

---

### Precedence is important, look:


    checkNotExists "this protocol already exists"
        >=> createProtocol
        >=> checkIntegrity "data integrity error, there are parameters with incorrect protocol information"
        >=> prepareChanges
        >=> Utils.applyChanges "unable to create protocol"
        >> Utils.log "created protocol" "protocol not created:" <| par


***

&& + composition = >=>

***
    module ROP =

        type Result<'a,'b> =
            | Success of 'a
            | Failure of 'b

        let (>=>) f g x =
            match f x with
              | Success s -> g s
              | Failure f -> Failure f

***

### Use case

***


#### Shared resource

- ProtocolInformation.xml (14M)
+
- ~60 .resource files

***

##### that needs to be updated safely

- no concurrency
- and no conflicts
- and all the files at once.

***

### Solution

Single api to update the system that provides **optimistic concurrency**  + **error reporting** in the failing case.

***

##### The ideal case would be to be able to

1. Check if there is any concurrency issue
2. Validate interity of the data
3. Validate integrity of the translations
4. Update the data
5. Log everything that happened or failed and report errors

***

### Implementation

From an implementation point, the solution materializes in a SPA application
that actually is a thin layer of a WebAPI.

***

## Enter F# magic

---

    open ProtocolsEditor.Application.UseCases
    open ProtocolsEditor.Infrastructure.Utils

    [<RoutePrefix("api/protocols/{boardTypeId}/{boardSubTypeId}")>]
    type ProtocolController() =
        inherit ApiController()

        /// Replaces the current protocol with the posted protocol
        [<HttpPut>]
        [<Route("")>]
        member x.Put(t: int, s: int, ps: Protected<Api.TranslatedProtocol>) =
            Protocols.updateParameters t s ps |> Utils.asResponse <| x.Request

---

    let updateParameters (t:int) (s:int) (tps:Protected<Api.TranslatedProtocol>) =

        checkHash "concurrency error, someone changed the same protocol while you were editing it"
        >=> checkIntegrity "data integrity error, there are parameters that are not from this protocol"
        >=> validateIdentifiers
        >=> checkTranslationKeyNotDefault DEFAULT_TRANSLATION_KEY (sprintf "%s is not a valid TranslationKey" DEFAULT_TRANSLATION_KEY)
        >=> checkTranslationKeysUniques "duplicated TranslationKey in this protocol"
        >=> prepareChanges
        >=> Utils.applyChanges "unable to apply changes"
        >> Utils.log "prot updated" "prot not updated:" <| (t, s, tprot)

***

### Domain Model

(images/class-diagram) == record types

***

    namespace ProtocolsEditor.Domain.Model

    module Protocols =

        open System
        open System.Xml.Serialization

        type ProtocolInformation = {
            LastUpdate: DateTime
            Parameters: ProtocolParameterInformation list
            Sensors: ProtocolParameterInformation list
        }

***

    namespace Definitions.Common.Domain
    {
        using System;
        using System.Collections.Generic;
        using System.Xml.Serialization;

        [Serializable]
        [XmlRoot(Namespace = "www.foobar.com/protocols")]
        public sealed class ProtocolInformation
        {
            public ProtocolInformation()
            {
                this.Parameters = new List<ProtocolParameterInformation>();
                this.Sensors = new List<ProtocolParameterInformation>();
            }
            [XmlElement]
            public DateTime LastUpdate { get; set; }

            [XmlArray("Parameters")]
            [XmlArrayItem("Parameter")]
            public List<ProtocolParameterInformation> Parameters { get; set; }


            [XmlArray(ElementName = "Sensors")]
            [XmlArrayItem(ElementName = "Sensor")]
            public List<ProtocolParameterInformation> Sensors { get; set; }
        }
    }

***

    type ProtocolParameterInformation = {
         Behavior: int
         BoardSubtypeId: int
         BoardTypeId: int
         DefaultGraphicRender: int
         FilterDefinitionInformation: FilterDefinitionInformation option
         IsComputed: bool
         IsPVPortalComputed: bool
         MainParameter: bool
         Mode: int
         ParameterOutputMode: int
         ProtocolParameterIdentifier: int
         PVPortalComputeOperation: int
         Tipus: int
         TranslationKey: string
         TranslationResourceName: string
         ViewModeType: int
    }

***

    type FilterDefinitionInformation = {
         CoefficientEvaluation: int
         ComputedReferenceClassName: string
         DailyGrouping: int
         DailyPeriod: int
         DecimalPlaces: int
         GraphicRender: int
         InstantaneousGrouping: int
         InstantaneousPeriod: int
         InvalidValues: int list
         IsSpecialFormat: bool
         MainCoefficient: double
         MeasureUnit: string
         MonthlyGrouping: int
         MonthlyPeriod: int
         ParameterSubType: int
         ParameterType: int
         SecondCoefficient: double
         SpecialFormatMode: int
         TranslationTableInformation: TranslationTableInformation option
         YearlyGrouping: int
    }

***

    type TranslationTableDetailInformation = {
         Key: double
         TranslationKey: string
    }

    type TranslationTableInformation = {
        TranslationResourceName: string
        TranslationTableInformations: TranslationTableDetailInformation list
        TranslationTableTypeFullName: string
    }

***

    type Board = {
        BoardTypeId: int
        BoardSubtypeId: int
        ParameterIds: int list
    }

***

### Theory

- Inmutabilty by default (actually, no `mutable` field used in this editor!)
- Functional

***

### Practice

- You need to mutate things: Lenses
- You need some objects

***

### I <3 F#

![F#](http://fsharp.org/img/logo/fsharp256.png)
