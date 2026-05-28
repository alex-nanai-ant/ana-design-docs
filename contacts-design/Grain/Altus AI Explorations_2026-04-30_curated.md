# Meeting name: Altus AI Explorations 4/30/2026

## Transcript

**Puru Agrawal:** Do you have any progress on the contacts piece?

**Petro Darii:** Yep, that's the next thing. So I took taxonomy and I extended that. So currently here is mocked data so I tried to play with that. So we have same entities concept here we have selection for type of entity like person of organization, same description aliases we have same connections so label description and for entities type like person type we have roles, some kind of predefined roles. It's multi selection like whatever we need and we have alicence so we can say Petrodari also Petro Peter Pidari roles, something like that.

**Puru Agrawal:** So how do I see things like email addresses for the person?

**Petro Darii:** You can add that. Currently no email addresses we can add that to like input or text area.

**Puru Agrawal:** Right. So let's say we have green recordings, emails, confluence pages, etc. Right. And so in these things participants show up and so there is some sort of a overall list of.

**Puru Agrawal:** These are the participants that show up and. P. Agrawal2@yahoo.com and puru.agrawaltisnova.com and,

**Puru Agrawal:** You know, whatever ID from Confluence. All map to Guru Agarwal, who is with Altus Nova. Right. And in the role of consultant@ISPN.

**Petro Darii:** So all those names will be aliases for main entity.

**Puru Agrawal:** Yes, but how do I pull that together?

**Petro Darii:** That's. So you want to manually pull that together or you want to backend find those entities.

**Puru Agrawal:** And I think the backend knows that all of those things exist. So there should be some list. Right. That I'm picking from.

**Puru Agrawal:** And then from that list I should be able to build out this context map.

**Petro Darii:** So you want list of all similar entities and you want manually to pick them and group them.

**Puru Agrawal:** Yeah. So I guess when we were talking about leveraging what we did with taxonomy, in my mind I was still imagining that we'd have this list of all of the identities that showed up somewhere. And then I could start to organize that list into contacts. And the contacts have roles, descriptions, aliases, et cetera. Right.

**Petro Darii:** Yeah. In previous implementation, like in previous poc, we had this kind of grouping, so we have possibility to group contacts. So do you want to see those contacts at least here on the left, or do you want to have, I don't know, dialogue for that extra window or on the right panel, like. Because in first implementation I had that concept.

**Puru Agrawal:** Yeah.

**Puru Agrawal:** I mean, so if you had.

**Petro Darii:** Like this. Yeah. Like with grouped contacts with different sources.

**Puru Agrawal:** So the primary thing is the list of contacts. Right. And so for each contact, we want to have a name, we want to have the role, we want to have the company, we want to have a description, and we want aliases.

**Puru Agrawal:** And the aliases would be email addresses, IDs, whatever that that person shows up as. Right. And so somewhere I have this list of all of the identifiers that show up. If I take the first one and drag it here, it creates. Drag it to a new row, it creates the.

**Puru Agrawal:** And then I can add information about the role, the company, the description, etc.

**Puru Agrawal:** And then this one somehow shows as highlighted, meaning it's assigned to this contacts list. And then if I take other ones and also drag them into the same thing, then it just adds them to the alias. Right. And so eventually all of the identities that show up in this list have been assigned to a contact that I've configured.

**Petro Darii:** Okay, so you want to do that manually. My proposal, that should be done on the backend side because we have information about those contacts and we are able to group them. Like, in my mind will be cool to do less things here on the front end with grouping, like only in some very.

**Puru Agrawal:** How do you know on the back end that Puru Agrawal, the contact and this individual who joined a grain recording, whose Identity was P. Agrawal2yahoo.com and this other individual who joined a grain recording on their phone and their phone number was 512-694-5346 and this other individual who forwarded an email yeah, like, are the same person. But those are all pretty normal things that we deal with. Like, if we go and look at all of the content from cssi, we're going to see all kinds of individuals that in general there's no information about.

**Andrey Orlenko:** Okay, your idea is that our system somehow in the back end, in the background, will identify this, contacts, names or emails, whatever additional information, and then on that contacts screen we drag and drop them. Like you as a user who knows this, all these persons, you kind of sort out that information, is that correct?

**Puru Agrawal:** Yeah, because all we know is the ID of the Confluence creator page, the ID of commenters, the, you know, whatever grain shows as the participant ID of various participants, the email address, maybe with some metadata of somebody who shows up in emails or forwarded emails. Like how does the system know who's who and what roles they have?

**Petro Darii:** We can automatically group like similar names, emails or something, and we can mark as attention needed other entities. For sure. Yeah.

**Puru Agrawal:** So like if, if you have these identities that are found, you can present them, right, in some sort of organized way to a person who then has to classify those identities. But like, how does the system know how to classify those identities?

**Andrey Orlenko:** System can only guess, of course, if, if only if, like, if Warren shows up, it can deduct, for example, if you run the bunch of some scripts, you know, and chats through LLM and for example, ask it to try to determine who is who and who is matching to who. Like that PR Graval is same as Peru, right. Then it will do maybe a decent job. But still the final approval of this is on the user side, of course. Right.

**Puru Agrawal:** So then is the metaphor that the user is the one responsible for creating this directory and AI might make recommendations about what's in there. Or is the metaphor that the AI creates this directory and then the user has to go back, read, review and make edits? Like, my concern would be that users become sloppy, don't bother to review anything that the AI has put together. And there's all kinds of mistakes in there, right? And then especially like when you're starting with a new client, there aren't tons of recordings and content and stuff to be able to. To say, like for the AI to make deductions of who's who. Right.

**Puru Agrawal:** So I think like, the reason we were talking about pulling taxonomy in is not to use like the visual taxonomy metaphors, but because taxonomy already supports this notion of description of aliases, etc. And so it's already like going into the prompt to say if you see X, Y or Z show up. Right. Then it really means A, B, or. Cuz like if we look at cssi, how many interactions have we had with folks? Like, I would imagine there's close to 100 different individuals.

**Puru Agrawal:** Right. And so like these five,

**Puru Agrawal:** It just feels like in a real project you're not going to have five, you're going to have a hundred. Like look at cssi, look, look at all the analysts, the sales reps,

**Puru Agrawal:** the consultants, people from Ops Pro, Debbie Krupitzer, you know, etc. Etc. That have kind of come and gone through that organization. Right. That will all show up in our

**Puru Agrawal:** space.

**Andrey Orlenko:** Yep.

**Petro Darii:** If it makes sense to reuse taxonomy editor for those as if you have 100 entities here, I'm not sure it will be very useful to have all this. Right.

**Puru Agrawal:** So I think, I think you're right. Like the idea would not be to use the visual layout editor. The idea would just be to use the same constructs around name description, aliases, and then the corresponding prompts to refer to that list. Right. And to kind of reinterpret anything that shows up accordingly.

**Petro Darii:** Okay, in that case, can I go back to my concept with those? Okay, maybe then let's try to discuss this concept again. Like with all, all entities here on the left. Like all entities from all different. Type of sources, we can group them. Let's say Alice Johnson and Alice Johnson. We will see a group at grouped group of contacts. Yeah, like same context with Alice's. As I understood, I need to add description and name here. We can add roles, what else? And so do we want like entities for Altus Nova or like.

**Puru Agrawal:** Yeah, so we have to see like the, the aliases, right?

**Petro Darii:** Yeah, yeah, we will see aliases here. I will add same thing.

**Puru Agrawal:** Like, yeah, so we should see the company, the role or roles, the aliases, a description and then like the official name. Right?

**Petro Darii:** Yeah. And, and like this are like those are aliases. So Alice Johnson and Alice Johnson are different entities, but they are alices. So they are grouped together in this case.

**Puru Agrawal:** Right. And so wouldn't it be better to see a single Alice Johnson with the phone, the various known emails, the various known aliases, the various known roles. Like

**Puru Agrawal:** to me, the fact that there are seven different identities for Alice Johnson doesn't change her role or her company or her department. And so having to define those seven times and group them together feels awkward.

**Petro Darii:** Yeah, but we're defining them only once. So currently, like I will simplify this ui, I will leave one name and maybe show somewhere like alias, alias, name and I will group emails, phones like and maybe sources somewhere. But role is one.

**Puru Agrawal:** So currently why would we see a phone number for each of those aliases?

**Petro Darii:** Yeah, yeah, yeah, I will group that. So it was like initial poc. Yeah, like theoretically they can have different phone, but in real life probably they will. One of them will not have phone or maybe there will be no phones at all.

**Puru Agrawal:** Yeah, like in our system, how would they show up? I guess if it's a grain participant who only called in by phone, then we might have a phone number. Right? Yeah, but then the phone is the alias of the individual.

**Petro Darii:** Yeah. And we will have this like entity here on the left with phone only and we will have grain icon. So. And we will manually select it. Like if you want to link it with somebody, we will select this form, entity, other entity, group them and we will have group.

**Puru Agrawal:** So then probably on this left side we should have two sections like assigned entities and unassigned or assigned identities and unassigned identities. And the goal becomes to have as few unassigned identities as possible. So you're periodically monitoring for any new unassigned identities that show up. And if you see one, then you should either assign it to an existing contact or set up a contact for it.

**Petro Darii:** Okay, like unassigned entity. What does it mean? It doesn't have name or like.

**Puru Agrawal:** So let's say I start a project and I see an individual called Petro darialtisnova.com show up in a grain recording. So then I create a contact for Petrodari and assign that identity to that contact. Okay. So you want a week later I see another Identity Show up, petrodarimail.com. right. But that new identity, which maybe showed up because of some Confluence page that petro Darimail.com created or whatever, is not assigned to a contact. And so I would need to be able to see like, oh, these new identities showed up. Either I should assign them to existing contacts or create a new contact to represent them. Right. But it's like if you have five different different identities because sometimes you dial in by phone, you have these different emails, etc, etc, they're unlikely to all show up right at the beginning. So you have a case like CSSI where it's an existing project and we have all this information, but the project's been going on for a couple of years. You have a case like Aquila commercial which is literally just about to start and it's going to be brand new stuff and new people showing up for the first time every other day. Right, yeah.

**Petro Darii:** So as I understood then we will have like separate thing as contacts. Yeah. So we will be able to create new contact here and let's say Petrodari and assign multiple not assigned entities to that contact.

**Puru Agrawal:** Identities. Identities.

**Petro Darii:** Yeah, yeah.

**Puru Agrawal:** Because there's only one entity that's.

**Petro Darii:** Yeah, yeah. I mean like abstractly, all those will be entities. Yeah.

**Puru Agrawal:** So let's think about what is the primary view like? In my mind the primary view is the list of contacts, right? And then if I select a contact, maybe there's an ancillary view of the activity, known activity for that contact. But really the activity is just a detail. It would be overwhelming to look at activity as the primary thing.

**Petro Darii:** So I can change layout with this view. Like I tried to reuse same layout as everywhere. Yeah, like left column, right column. But we can change it and use like main view.

**Puru Agrawal:** I think you would still have the left, right and center. It's just what is the detail? The detail to me is activity that may have occurred from that contact. But the primary information is the name, role, department description and known aliases.

**Petro Darii:** Okay, so. So you want to swap some places? Yeah, like yes, middle. Okay, okay, got it. Right.

**Puru Agrawal:** And then the list of contacts. I think we'd want to be able to maybe not on day one, but eventually sort and search. Because if you end up with hundred, so there, like are you searching the identities or are you searching the contacts?

**Petro Darii:** We can filter everything like together, like both.

**Puru Agrawal:** Okay, like why not?

**Petro Darii:** It's only titles and it's very fast. It's on front end side.

**Puru Agrawal:** It's like once we have this information, then the question becomes how is that made available to the AI when it's processing information? Right. And so like for taxonomy we push that information into the system prompt.

**Puru Agrawal:** And so for contacts, are we doing the same kind of thing or are we having some reference doc that the system that we point the system to, right? That says hey, these are the people and blah blah, blah. And so then like in my

**Puru Agrawal:** sessions, if I'm doing some analysis that says disregard input from Puru Agarwal, like, you know, what are the three most important things for ISPN in 2026? Disregard input from Altus Nova because we don't want our own statements to be self reflected. Reinforcing to Anna then what is the format in which it knows how to disregard input from Altus Nova people? Is it the directory coming in as the system prompt? Is it some page that it knows? Is the directory that it cross references? It's indexing Again, like that's probably a question for Andrew.

**Andrey Orlenko:** System prompt is most straightforward.

**Petro Darii:** What I was thinking same as for taxonomy. Correct.

**Puru Agrawal:** But do we have any concerns about the system prom becoming too large and bloated?

**Andrey Orlenko:** I think I don't know. Of course it depends on the scale. But

**Andrey Orlenko:** for what we see here, like let's say you have up to 100 contacts for a client. Yeah, that's a lot. But.

**Andrey Orlenko:** We need to test this. Another option would be so yes, most straightforward is system prompt. Another option would be have a new skill or new tool, technically speaking that is able to receive all the contacts for our chat and so it will determine when it needs to understand something about contacts. So it will be not there on every your question. Yeah, so the question I think boils down is do you want to have Anna always this context of every contact or you not? If you not, then we do it as a tool and Anna will know when to call this tool. If yes, then it's better to just have it in system prompt.

**Puru Agrawal:** Like I would imagine that for most inquiries you don't care, but for certain inquiries you're going to say either disregard input from this individual or people from this organization or whatever, or you're going to say I'm looking for information from that individual. Right. So did Warren Dasio ever discuss blah, blah, blah and yeah, Warren Dasio has five different identifiers in the system and we want it to go and know that. Okay, Anything from any of those identifiers that matches that query is relevant. So I could see us saying exclude and I could see us saying include and search for Right. And they would be either at the individual level, the role level, the company level,

**Petro Darii:** does it make sense to save it as file and add to index?

**Puru Agrawal:** So I think I don't know the answer. That's the question. And then. Like it has to be treated as a high priority item.

**Andrey Orlenko:** I don't know. I would prefer it a separate tool so that it knows when to. When to call it. Yes, it's.

**Alex Nanai:** It's. It's definitely too. I mean, to me it looks so. When Andre said it makes the most sense, once it spots any name, it can call the tool.

**Puru Agrawal:** So maybe like just the list of contacts is in the system prompt and then all the details and aliases and stuff are accessed using the tool.