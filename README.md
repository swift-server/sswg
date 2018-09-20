Swift Server Work Group (SSWG)
--
The Swift Server work group is a steering team that promotes the use of Swift for developing and deploying server applications. The Swift Server work group will:

Define and prioritize efforts that address the needs of the Swift server community
Define and run an incubation process for these efforts to reduce duplication of effort, increase compatibility and promote best practices
Channel feedback for Swift language features needed by the server development community to the Swift Core Team
Analogous to the Core Team for Swift, the work group is responsible for providing overall technical direction and establishing the standards by which libraries and tools are proposed, developed and eventually recommended. Membership of the work group is contribution-based and is expected to evolve over time.

The current Swift Server work group consists of the following people:

* Chris Bailey (@chris_bailey, IBM Kitura)
* Ian Partridge (@ianpartridge, IBM Kitura)
* Logan Wright (@loganwright, Vapor)
* Tanner Nelson (@tanner0101, Vapor)
* Johannes Weiss (@johannesweiss, Apple)
* Tom Doron (@tomerd, Apple)
* Steve Algernon (@salgernon, Apple)

Community Participation
--
There isn’t a distinction between stakeholders and contributors for this effort. Everyone is welcome to contribute in the following ways:

* Proposing new libraries and tools to be considered
* Participating in design discussions
* Asking or answering questions on the forums
* Reporting or triaging bugs
* Submitting pull requests to the library projects for implementation or tests

These conversations will take place on the [Swift Server forum](https://forums.swift.org/c/development/server). Over time, the work group may form smaller teams to focus on specific technology areas.

Recommended Server Libraries
--
The main goal of the Swift Server work group is to eventually recommend libraries and tools for server application development with Swift. The difference between this work group and the Swift Evolution process is that server-oriented libraries and tools that are produced as a result of work group efforts will exist outside of the Swift language project itself. The work group will work to recommend libraries and tools as they move into their release stages.

Incubation Process
--
The details of the process under which libraries and tools become recommended still needs to be formalized. One of the first goals of the Swift Server work group is to define this process. For now, the general idea is that it will take a federated approach with the following conditions:

* Swift Server work group will define requirements for each stage of a package’s incubation from definition to recommendation
* Swift Server work group will maintain and publicize a list of packages with current incubation status
* Libraries and tools will be developed as packages with Swift Package Manager
* Packages can live in any publicly accessible open source repository and should adopt Apache License v2

The Swift Server work group will direct the incubation of new server-focused libraries and tools through four phases: Determine Need, Proposal, Prototyping and Development, and Recommendation.

Determine Need
--
To start this process, the work group first determines and maintains a prioritized list of what’s needed for development and deployment of Swift server applications. The purpose of this list is to seed the community with ideas that will help solicit the creation of proposals.

Proposal
--
As proposals are brought forth by the community, the work group will evaluate them against the following criteria:

* Does it provide a reasonable set of capabilities and functions
* Are the capabilities and functions useful
* What types of applications would benefit from them
* How would the capabilities and functions be used
* What is the intended approach to implement them
* A proposal is then reviewed and socialized with the wider Swift community as a “Pitch” in the Server forums, similar to how Swift Evolution handles them.

Prototyping and Development
--
Once a proposal is approved by the work group and input has been received from the wider community via the [Swift Server forum](https://forums.swift.org/c/development/server), prototyping and development can begin. The work group and the greater Swift Server community will work to sponsor and promote incubation proposals, but once approved, libraries and tools can be prototyped and developed independently of the group.

Once the full set of functions has been developed, released, adopted and feedback acted on, the work group will branch a formal recommendation process.

Recommendation
--
As mentioned earlier, the Swift Server work group will be defining a formal process to determine how a library or tool will be recommended. Watch this space for more information as the details develop. Currently SwiftNIO is the only recommended library by the Swift Server work group.

Forums
--
The Swift Server work group uses the [Server](https://forums.swift.org/c/development/server) forum for general discussion.
