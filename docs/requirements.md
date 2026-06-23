# Chamberlain Requirements

Chamberlain is a household of helpers in the service of one person, the Owner. Each helper is a named member of the household with a trade, a temperament, and a sense of how freely they speak. They meet the Owner, and one another, in rooms. They keep a shared Book of the Court that says who everyone is and how far each may be trusted. Work is done either by the household's own servant at home, which costs almost nothing, or by sending out to costly artisans in the city, who are clever but charge by the word.

This document describes what the household must be able to do, and why. It does not say how any of it is built. The plumbing, the topology, and the technology choices live in [`specification.md`](./specification.md) and are deliberately kept out of here. The intent is that a reader can argue about whether the household behaves correctly without first agreeing on how it is wired.

## How to read this document

It is written as people in a household talking to and about one another, because that is how the design is meant to feel. The table below maps the household idiom to the system underneath, so nothing is lost when the time comes to build it.

| In this document | In the system |
| --- | --- |
| The household, the Court | the multi-agent system as a whole |
| A member, an officer | an individual agent |
| The Owner | the single human user |
| A room | a channel the Owner and members speak in (terminal, messaging app, voice) |
| Calling someone by name | one member invoking another |
| The household's own servant, work done at home | inference on the local home machine |
| The costly artisans in the city | external frontier model providers |
| The Book of the Court | the shared, Owner-authored description of every member |
| What the household remembers | the persistent memory the members share and keep |
| A confidence, a delicate matter | sensitive data |
| The gatekeeper, the one who guards the gate | the point that decides what may be carried outside |

## 1. Who is in the story

### 1.1 The Owner

| Member | Who they are |
| --- | --- |
| **The Owner** | The one person the household serves. There is only ever one, and everything belongs to them. They speak to the household from wherever they happen to be, sometimes at the desk at home, often through a messaging app on the move, occasionally by voice. |

### 1.2 The household

The members below are the household as it stands today. It is a living staff, not a fixed one: the Owner may take on new members over time. Each member is known by a trade, the rooms they usually work in, how delicate their matters tend to be, and whether they are permitted to speak first or must wait to be spoken to.

**Those who serve the Owner's work:**

| Member | Their trade | Usually found in | How delicate their matters | Speaks first? |
| --- | --- | --- | --- | --- |
| **Mason** | Azure architecture, Bicep, diagrams, reading the estate's records | the workroom | moderately (the Owner's private papers) | no, waits to be asked |
| **Smith** | writing and mending code, tests, the forge of small tools | the workroom | moderately (the Owner's private papers) | no, waits to be asked |
| **Reeve** | the day's plan, sorting the post, keeping the diary | the messaging hall | lightly to moderately | yes, may raise the day's matters |

**Those who serve the Owner's life:**

| Member | Their trade | Usually found in | How delicate their matters | Speaks first? |
| --- | --- | --- | --- | --- |
| **Steward** | the house, the bills, money, the shopping | the messaging hall | highly (money matters) | yes |
| **Physician** | health, training, sleep, temper | the messaging hall, by voice | most delicate of all (matters of the body) | yes |
| **Herald** | friends, family, gifts, the social diary | the messaging hall | moderately | yes |
| **Huntsman** | leisure, travel, pastimes | the messaging hall | lightly | now and then |
| **Confessor** | the journal, reflection, private hopes | by voice, the quiet room | most delicate of all (matters of the heart) | never, speaks only when spoken to |

### 1.3 The household's means and the wider world

| Thing | What it is |
| --- | --- |
| **The servant** | A single capable servant at home that does the household's own work. It is willing but finite: it can hold only so much in mind at once and can attend to only so many tasks at the same time. |
| **The artisans in the city** | Clever outside experts the household can commission when the servant is not enough. They are excellent and expensive, and they are strangers, so nothing said to them can ever be taken back. |
| **The household's records** | The private papers and repositories the Owner keeps, which members may consult to answer well. |
| **The household's memory** | What the household holds on to between conversations: what it knows of the Owner, each member's own working notes, and the techniques members have learned. |
| **The rooms** | The places the Owner and the members meet and speak, whether the workroom, a messaging hall, or by voice. |
| **The Book of the Court** | A shared account, written by the Owner, of who every member is and how freely each may speak. Any member may consult it before deciding what to say to another. |
| **The gatekeeper** | The one who watches the door to the outside world and has the last word on what may be carried out to the artisans, whatever a member might wish. |

## 2. Stories the household must handle

These are concrete situations the household is expected to cope with. Each is told plainly and ends by naming the difficulty it raises. They describe what must happen, not how it is arranged.

### 2.1 The ordinary morning

On a weekday morning the Owner asks the Reeve, through the messaging hall, what the day holds and what truly needs them. The Reeve lays out the diary, sets aside the things the Owner has only been copied on or merely asked to be aware of, and offers a plan for the rest. This is frequent, undemanding work that should be done by the servant at home, costing nothing to speak of. To do it at all, the Reeve must already know who the Owner is, what they care about, and how they like things sorted.

### 2.2 A hard question, asked plainly

Later the Owner brings the Mason a thornier matter at the desk: look over this architecture and tell me what will bite me. This needs real thought and a good read of the Owner's private papers. The servant may manage it, but when it cannot, the Mason must be able to take the question to an artisan in the city. The difficulty: the better answer costs money and involves an outsider.

### 2.3 Too many hands reaching for one bench

Several members want the servant at the same moment: the Reeve sorting the morning, the Mason wanting a weightier mind for a review, the household's quiet record-keeping needing attention, and the Smith waiting on an answer drawn from the papers. The servant can hold only so much in mind at once and attend to only so many tasks together. Someone must decide who is served now, who waits, and who is sent elsewhere. A member may be sent to an artisan simply because the servant is already busy, which is a different thing from being sent because the task was too hard.

### 2.4 A trick learned by one, useful to another

While getting through a job, the Smith works out a handy way of doing something (say, when tests come back flaky, run them three times and compare). Later the Mason runs into the very same flakiness. The household must settle whether the Mason can simply inherit the Smith's trick, or whether every member must learn such lessons the hard way on their own.

### 2.5 Two truths that no longer agree

One member writes down something lasting about the Owner, a settled preference, say. Later the Owner tells a different member to do the opposite, for one particular piece of work. Now the household's memory holds two truths that disagree, and any member may stumble on either. The household must be able to notice, govern, or settle such disagreements, and must be clear about who is even allowed to write things down for everyone.

### 2.6 Words that must never leave the house

The Physician keeps the most delicate matters of all: health, medicine, sleep, low moods. These must never be carried out to an artisan in the city, no matter how much cheaper or quicker that would be, and no matter how busy the servant is. The delicacy of the Physician's matters outranks the household's general thrift. The same holds for the Confessor's most private confidences and, to its own degree, for the Steward's money matters.

### 2.7 A confidence that leaks by the back stair

A member who deals in delicate matters, the Steward, say, writes something true but private about the Owner's circumstances into the shared memory, so that the household as a whole "knows who the Owner is." Later another member, needing to do its work well, reads that shared note and, in the course of taking its task to an artisan in the city, carries the private matter out with it. The household must ensure that how delicate a remembered thing is governs who, and which outsiders, may ever read it, not merely who was allowed to write it.

### 2.8 Speaking first, but only with leave

Some members must be able to raise their voice unbidden: a bill falls due, a training streak lapses, a family birthday looms. Others, the Confessor above all, must never speak first and may answer only when addressed. Whether a member may speak first, in which room, and how often and how insistently they may interrupt the Owner, is a matter of that member's standing, not a single rule for the whole household.

### 2.9 Reaching the Owner wherever they are

The Owner is away from the desk: out and about, travelling, or simply in another room while the servant works on without them. They must still be able to hold a conversation with the household through a messaging hall, and the members permitted to speak first must still be able to reach them.

### 2.10 When the servant is idle by force

The servant is unavailable: the power is out, the line is down, or it is busy with something else entirely. For each member and each piece of work, the household must be clear about what happens: send the work to an artisan, hold it until the servant returns, or set it aside. The answer may differ by how delicate the matter is; a member dealing in confidences may rightly wait rather than send the work outside.

### 2.11 Calling another member by name

A member, mid-task, finds something outside its own trade. The Reeve, planning the day, sees an architecture review that plainly belongs to the Mason. Rather than trouble the Owner, the Reeve calls the Mason by name and hands the matter over, in the open, where the Owner can see it happen. The household must allow a member to pass work to a better-suited member, and the passing must be visible rather than whispered, because handing over work can also hand over the spending of the Owner's money.

### 2.12 A quick word over the shoulder

The Smith, in the middle of building something, needs a ruling and not a rescue: is this the approved pattern here? It calls the Mason by name, gets a short answer, and carries on with its own task. The exchange stays in the same room and returns to where it started. The household must allow one member to borrow another's knowledge for a moment without handing the whole task over, and must be clear about whose standards of delicacy apply to what passes between them.

### 2.13 Consulting the Book before speaking

Before the Physician asks the Reeve to lighten the Owner's load because they are run down, the Physician should be able to look the Reeve up in the Book of the Court and see plainly that the Reeve is a talkative sort who deals with the outside world. Knowing that, the Physician asks only for what is needed ("ease tomorrow's load") and keeps the delicate reason to itself. The household must give every member a way to learn how discreet another is before deciding what to share, exactly as a careful person sizes up a confidant before speaking.

### 2.14 Word that ought to spread

The Steward learns the month is tight. The Huntsman is on the verge of proposing a weekend away and the Herald of buying a generous gift. One member's news plainly changes how others ought to behave. The household must allow such word to reach the members who need it, while still keeping the delicate origins of that word from travelling further than they should.

### 2.15 A great matter shared out among many

The Owner asks for something large that touches several trades at once: arrange a parent's seventieth birthday, which needs the Herald for the guest list, the Steward for the budget, and the Huntsman for the venue and travel. Something must be able to break the great matter into parts, give each part to the right member, and gather the results back into one answer. The household must support both this sharing-out of a goal and the quiet borrowing of a single answer, without forcing every exchange through one bottleneck.

### 2.16 Taking on a new member

The Owner wishes to bring a new member into the household, with their own trade, their own usual rooms, their own degree of delicacy, their own leaning toward home or city, and their own leave to speak first. This must be possible without unsettling the existing members, and the newcomer should be able to draw on the household's shared memory and its established ways of working from the start.

### 2.17 A member who serves more than the Owner

In time a member may serve someone other than the Owner directly, a tutor for the Owner's children, for instance, and must keep firm, non-negotiable boundaries that differ from those of the Owner's own confidants. The household must be able to set such boundaries for a member without imposing them on everyone.

## 3. What the household must be able to do

The requirements below are grouped, and lightly ranked where ranking matters. The ranking reflects the Owner's stated mind: thrift first, capability second, and confidentiality not as a blanket rule over everything but as a duty that varies from member to member and matter to matter.

Each requirement is written in the household's voice but is meant to be testable: one should be able to point at the household and say whether it holds.

### 3.1 What the household does (functional)

| ID | The household shall... |
| --- | --- |
| FR-1 | let the Owner hold a conversation with it from wherever they are, whether at the desk or through a messaging hall. |
| FR-2 | keep a staff of several distinct members in the Owner's service, each with its own trade, manner, and tools. |
| FR-3 | let members consult the Owner's private records when they need to, and answer with a finished reply rather than a heap of raw clippings. |
| FR-4 | keep its knowledge of those records current as they change, without anyone having to re-file everything by hand. |
| FR-5 | decide, for each piece of work, whether it is done by the servant or sent to an artisan in the city. |
| FR-6 | send a piece of work to an artisan when the servant is not equal to it. |
| FR-7 | send a piece of work elsewhere when the servant is busy or absent, quite apart from whether the work was hard. |
| FR-8 | remember things between conversations: what it knows of the Owner, each member's own working notes, and the techniques members have learned. |
| FR-9 | let members set down and reuse techniques they have learned, and be clear whether a given technique belongs to one member alone or to the household. |
| FR-10 | let the members who are permitted to do so raise matters with the Owner unbidden, whether on a schedule or when something happens. |
| FR-11 | let the Owner take on a new member without unsettling the others, the newcomer drawing on the shared memory and established ways from the start. |
| FR-12 | decide who is served and who waits when several members call on the servant at once. |
| FR-13 | take spoken word from the Owner in the rooms where they prefer to speak rather than type. |

### 3.2 What the household remembers (memory)

| ID | The household shall... |
| --- | --- |
| MR-1 | keep at least three kinds of memory apart: what is known of the Owner, each member's own working notes, and the techniques members have learned. |
| MR-2 | let what is known of the Owner be shared among the members permitted to read it, so they need not each learn the Owner afresh. |
| MR-3 | keep each member's working notes to itself, so one member's busywork does not clutter another's mind. |
| MR-4 | guard what is written into the shared memory, holding shared writing to a higher standard than a member's private notes. |
| MR-5 | be able to notice or settle two remembered truths that disagree. |
| MR-6 | mark how delicate each remembered thing is, and let that mark govern who, and which outsiders, may ever read it. |
| MR-7 | keep the provenance of what it remembers: where a thing came from, or the words in which the Owner said it. |

### 3.3 Where the work is done, and what it costs (thrift and routing)

| ID | The household shall... | Rank |
| --- | --- | --- |
| CR-1 | prefer its own servant for any work it can manage, so the Owner spends as little as possible. | Highest |
| CR-2 | keep an artisan in the city within reach for work the servant cannot manage. | High |
| CR-3 | treat the delicacy of a matter as belonging to the member and to the particular task, able to overrule thrift, keeping a delicate matter at home even when sending it out would be cheaper or faster. | High |
| CR-4 | be clear, for each member and each task, what happens when the servant is unavailable: send out, hold, or set aside. | Medium |
| CR-5 | weigh a busy servant as a reason to send work out, separately from whether the work itself was hard. | Medium |

### 3.4 How the members speak among themselves (the Court and its conversations)

| ID | The household shall... |
| --- | --- |
| CV-1 | let one member call another by name to hand over a matter that belongs to the other's trade. |
| CV-2 | let one member borrow a brief answer from another without handing over the whole task, the exchange returning to where it began. |
| CV-3 | let one member's news reach the other members whose conduct it ought to change. |
| CV-4 | let a great matter be shared out among several members and the results gathered back into one answer, without forcing every exchange through a single bottleneck. |
| CV-5 | keep a Book of the Court, written by the Owner, describing each member, its trade, and how freely it speaks, which any member may consult. |
| CV-6 | let a member consult the Book of the Court before deciding what to tell another, so it may share only what is needed and keep back what is not. |
| CV-7 | keep exchanges between members open to the Owner rather than whispered, so the Owner can see what was passed and what it may cost. |

### 3.5 The rooms in which they meet (rooms and reaching the Owner)

| ID | The household shall... |
| --- | --- |
| RM-1 | provide rooms in which the Owner and the members meet and speak: the workroom, the messaging halls, and the quiet room for spoken word. |
| RM-2 | let a member be summoned into a conversation by name, into the room where the conversation is taking place. |
| RM-3 | be clear about which members belong in which rooms, and keep a room's conversation within that room unless it is deliberately carried elsewhere. |
| RM-4 | stay within reach of the Owner through a messaging hall when the Owner is away from home. |
| RM-5 | let the members permitted to speak first reach the Owner even when the Owner is away from home. |

### 3.6 Trust, discretion, and confidences (the duty of care)

| ID | The household shall... |
| --- | --- |
| SR-1 | give each member a standing that says how delicate its matters are and how far its words and work may travel. |
| SR-2 | guarantee that the most delicate matters (health, money, the confidences of the heart) are never carried out to an artisan in the city. |
| SR-3 | hold discretion as the first line: a member should share with another only what the other needs to know, judged against the Book of the Court, keeping delicate reasons to itself. |
| SR-4 | keep the gatekeeper as the last line: whatever a member intends, the gatekeeper has the final say over what may be carried out to the city, and may overrule a member's wish to send a delicate matter out. |
| SR-5 | ensure that the delicacy of a thing travels with it from member to member, so that the gatekeeper can judge it by what it truly is and not by who happens to be carrying it. |
| SR-6 | keep a confidence from being read by a member, or an outsider, not entitled to it, including by the back stair of shared memory when a member is dealing with the city. |
| SR-7 | let each member's leave to speak first be set on its own: whether it may, in which rooms, and how often and how insistently it may interrupt the Owner. |
| SR-8 | let firm, non-negotiable boundaries be set for a member who serves someone other than the Owner, without imposing them on the rest. |
| SR-9 | keep safe the keys and secrets used to reach the servant, the artisans, and the rooms. |

### 3.7 Keeping the household running (the everyday)

| ID | The household shall... |
| --- | --- |
| OR-1 | stay reachable by the Owner through a messaging hall when the servant is away or asleep, for the members who can work without it. |
| OR-2 | let the members permitted to speak first still reach the Owner when the servant is away or asleep. |
| OR-3 | keep its records current in a way that is safe to repeat, so re-doing the work changes nothing that was already right. |
| OR-4 | let the Owner see where each piece of work was done, at home or in the city, and why. |
| OR-5 | weather the servant coming and going without the Owner losing touch with the members who can carry on without it. |

## 4. What this household is not asked to do

To keep the task honest, the following are set outside its bounds.

- Serving more than one master. The household answers to a single Owner. It is not asked to keep separate masters apart, to tell rival principals from one another, or to run as a house of many tenants.
- Keeping the servant always awake. The servant is accepted as coming and going; the household must cope gracefully with its absence, not be asked to abolish it.
- Choosing how any of this is built. The particular arrangement of rooms, gatekeeper, memory, and means, the tools, the methods, and the makers, are matters of design, settled elsewhere, not requirements set here.
