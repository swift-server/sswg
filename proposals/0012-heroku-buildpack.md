# Solution name

* Proposal: SSWG-0012
* Authors: [KyleF](https://github.com/kylef), [VZSG](https://github.com/vzsg)
* Sponsor(s): TBD
* Review Manager: [Logan Wright](https://github.com/loganwright)
* Status: **Implemented**
* Implementation: [Heroku Buildpack](https://github.com/vapor-community/heroku-buildpack)

## Package Description

This package is a buildpack that is used to host swift server projects on the [herokou](http://heroku.com/) platform. It includes things like swift versioning and general setup to accomodate swift.

|  |  |
|--|--|
| **Package Name** | `n/a` |
| **Module Name** | `n/a` |
| **Proposed Maturity Level** | [Incubating](https://github.com/swift-server/sswg/blob/master/process/incubation.md#process-diagram) |
| **License** | [BSD3](https://github.com/vapor-community/heroku-buildpack/blob/master/LICENSE) |
| **Dependencies** | [] |

## Introduction

Provide easy integration between swift server applications and heroku.

## Motivation

Although many people hosting on heroku are moving to Docker containers, and this might be recommended for more enterprise level projects, there are still many users still looking for the ease and familiarity of buildpacks. In the swift ecosystem this makes hosting quite easy and lowers barriers to adoption/deployment.

## Proposed solution

Integrate with heroku's buildpack system. For more integrated languages, heroku seems to provide their own buildpacks. For now we will need to provide our own.

## Detailed design

Already implemennted

## Maturity Justification

This package is already being used across the swift ecosystem to host many projects, it has been available and maintained in various forms since the beginning of swift linux compatibility, it has 20 committers from various affiliations, demonstrates regular updates associated with new swift releases, and generally satisfies the requirements of the sswg.

## Alternatives considered

There are other buildpacks available, but they seem less maintained or generic. While this package is hosted in the vapor-community org, it supports swift more generally and can even easily host a bare swift-nio server.

This buildpack is a fork originating with another author/organization.. we have [permission from the original author](https://github.com/kylef/heroku-buildpack-swift/issues/45) to continue with the fork in its current location and treat that as the head of the project.
