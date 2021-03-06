# LTEP summer project

Long term evolution of software product lines - Summer project forskerlinjen 2019

## About

This project is created by Ida Sandberg Motzfeldt and Eirik Halvard Sæther, in regards to [forskerlinjen](https://www.uio.no/studier/program/forskerlinjen-informatikk-1/). Forskerlinjen is a research program for bachelor students studying informatics at the University of Oslo, which gives young students experience as researchers. Most of the work of this project was done during the summer of 2019.

Software engineering methodologies for highly-variable software systems lack support for planning the long-term evolution of software. This project will try to aid in tackling some of the problems related to the long term evolution planning of software product lines, by developing a formal methodology to explicitly handle the evolution of a software product line, as well as specify temporal constraints for plan analysis. This plan is then verified by a model checker, to make sure the plan is producing a valid feature model at every (future) point in time. With a simple temporal grammar, the user can express constraints that can hold over the entire planned evolution. This is done by using structural operational semantics to formalize the evolution of the software product line, as well as using linear temporal logic to check temporal constraints. The Maude system is used to implement the specification.

This project is part of a larger collaboration between the University of Oslo and Technische Universität Braunschweig (TUBS), where the goal is to deal with issues in long-term evolution planning by developing and formalizing new methods. The collaboration will continue for two years, expanding on and adding to the work we have done this summer.

This repository consists of several Maude files, as well as a report.

## Overview

- `featuremodel.maude`: Modules containing the implementation of the feature models and the feature model evolution operations.
- `featuremodel-checker.maude`: Containing the module implementing the model checker for temporal constraints regarding feature model evolution.
- `featuremodel-test.maude`: Simple example of a valid feature model and some valid and non valid applications of operations to the feature model.
- `example.maude`: A full example including a valid feature model evolution plan, as well as tests for every temporal formula to test and give example usage to the different constraints.
- `example2.maude`:  An example of a feature model evolution plan that is not valid.
- `paradox1.maude`: Example of a non valid feature model evolution plan that is not valid. Running this will produce an error showcasing what went wrong in the execution of the plan.
- `paradox2.maude`: Another example of a non valid feature model evolution plan.

## Usage

If you are using a feature model to develop a software product line, you can use our system to verify that your plan is correct, and to see what your feature model will look like at a future time. To use the system, construct a Maude module on this form:

```haskell
load featuremodel-checker.maude

mod MY-PLAN is including FEATUREMODEL-CHECKER . protecting STRING + NAT .
    --- You can use String as the sort for FeatureID, GroupID, and Name.
    --- Could also have been Qid or Int (or anything else).
    subsorts String < FeatureID GroupID Name .
    --- Create initial configuration (what your feature model looks like now)
    op configuration : -> Configuration .
    eq configuration = FM("RootID",
                        ["RootID" -> f("RootName", noParent, noG, MANDATORY)])
                        # done
                        # empty . --- Example configuration.  
    --- Model plan (which changes you plan on making, and when)
    op plan : -> Plans .
    eq plan = at 1 do --- specify time for the plan, must be natural number
              --- add a group to the root feature
              addGroup("RootID", "Group1", AND)
              --- add a feature to the newly created group
              addFeature("ChildID", "ChildName", "Group1", OPTIONAL) ; ;;
              at 5 do
              --- Give the new feature a better name
              renameFeature("ChildID", "BetterName") ; .

   --- Use plan and configuration to make an initial state for the model checker
   op init : -> TimeConfiguration .
   eq init = currentTime: 0 C: configuration plan: plan .
endm
```

Next, run Maude and input `in <filename.maude>`. Next, check that your plan runs without errors:

```
Maude> rew init .
```

The result should be

```
rewrite in MY-PLAN : init .
result TimeConfiguration:
currentTime: endTime
C:
FM("RootID", ["ChildID" -> f("BetterName", "RootID", noG, OPTIONAL)]
  + ["RootID" -> f("RootName", noParent, g("Group1", AND, "ChildID"),
    MANDATORY)])
#
done
#
empty
plan: end
```

Here, we can see the final state of the feature model, with all the plans implemented. If it gets stuck, it will give an error message showing why, which can be used to correct the plan. If the plan is correct, like it is here, you can test temporal constraints:

```
Maude> red check(init, feature "ChildID" exists after 1) .
reduce in MY-PLAN : check(init, feature "ChildID" exists
    after 1) .
SUCCESS
result Bool: true
```

### Output of program

#### Soundness checker

Since maude produces a lot of information that are hard to parse and make sense of, we have included print statements across the code that will give you all the necessary information. Running the maude modules should be run with the following command:

```
maude -print-to-stderr 1> /dev/null <filename.maude>
```

The `-print-to-stderr` flag will send all the print statements to the stderr channel, which allows us to disregard the standard output (stdout). If the program ran without errors, no output will be produced. However, if the plan contains errors, the output will report what went wrong, at what timepoint, and what operation. The output can look something like this:

```
ERROR: Feature with ID pilot does not exist.
PLAN ERROR: 5 - addGroup(pilot, "pilot1", AND)
```

#### Model checker

Assuming a sound plan, you construct and check temporal statements using the model checker. See `example.maude` for a full example. Given an initial model `init`, we can check the temporal properties like this:

```
red check(init, feature car exists at 2) .
red check(init, neg feature distance exists at 2) .
red check(init, group "comfort1" exists at 4) .
red check(init, feature distance exists at 7) .
```

Each `check` statement will produce either `SUCCESS` or `FAILURE` print statements.

Running such a module with `maude -print-to-stderr 1> /dev/null example.maude` will produce the following output:

```
SUCCESS
SUCCESS
SUCCESS
FAILURE 
```

