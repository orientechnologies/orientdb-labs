OEP comments: https://github.com/orientechnologies/orientdb-labs/issues/1

**Summary:**

We need a structured process to discuss and approve new features and structural changes, internally and with the community.

**Goals:**

- Track the product evolution
- Have better awareness of all the team about new features
- Avoid introduction of inconsistent behaviors given by a partial points of view during implementation (without a proper logical review)
- Involve the community in the evolution process 
- Provide contributors and users some structured material to help and to understand where we are going.

**Non-Goals:**

- Change the final decision process/ownership on new features and product changes

**Success metrics:**

- Time to market (to be refined): the process does not have to be too slow and block the evolution and release deadlines
- N. issues for inconsistent behaviors

**Motivation:**

Today we have too many inconsistent behaviors, every time in introduce a new features we miss some basic concepts, we do not validate global consistency, we lack an impact analysis and ultimately we introduce regressions.

We also need a way to communicate with the community, involve users and new contributors, let people understand where we are going, and finally help them to help us

This comment describes very well the community point of view: 
https://github.com/orientechnologies/orientdb/issues/4806#issuecomment-225452814

**Description:**

This process is explicitly inspired by OpenJDK JEP process http://openjdk.java.net/jeps/1

All the new enhancements will be discussed in a defined process, called OrientDB Enhancement Process (OEP). 

`OEP <N>` also for `OrientDB Enhancement Proposal n. <N>`

The proposal is following:
- Use the current project issue tracker and source master branch for OEP
- Create an .md file in the master branch with the description of the OEP. The file name has to be `OEP_<N>.md`, eg. `OEP_15.md`. The .md file has to be updated every time there is a decision about a change in the OEP proposal. It also have to include an impact matrix describing how this OEP impacts single components of OrientDB.
- Create an issue for each OEP, that will be used to discuss 
- Only the core team (committers) should create and OEP, but all the users/community members should participate to the discussion
- Use issue numbers for OEPs, eg the issue n.1 is [OEP 1]
- The issue title has to be `[OEP <n>] <title>`
- The Issue description is the initial proposal of the OEP.
- There is a tempate for OEP, that should be used when opening the OEP issue and copied to the .md file as well.
- The community and the committers can comment and vote on the OEP
- The OEP should have an elapsed time, that has to be defined according to the proposed milestone date for release. When that time expires, the OEP has to be approved or rejected
- The core team is in charge of approving or rejecting an OEP

**Alternatives:**

Keep doing what we did before.

**Risks and assumptions:**

- The core team has to be committed on this, for every new feature, otherwise the process is completely useless and time consuming
