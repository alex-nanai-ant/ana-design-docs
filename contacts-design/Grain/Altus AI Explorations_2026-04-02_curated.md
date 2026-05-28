# Meeting name: Altus AI Explorations 4/2/2026

**Puru Agrawal:** So I had to redo this and I might have deleted the original session. Right. But it's basically saying this is validated by a stakeholder who says blah, blah, blah, blah, blah. But then that stakeholder is me. I have to tell it like, no, like I am putting on a wall and you cannot use my own statements as validation of what I'm saying to you now. I'm looking for other people's validation. Right. And so like we had this whole discussion.

**Puru Agrawal:** About this potential strategy and then I had to give it instructions.

**Puru Agrawal:** So this was on March 25th or whatever. And I had to tell it like, you know, ignore all comments from Puru Agrawal and only use content that's older than March 25th. Because March 25th is when this idea had first come up and like immediately, because we talked about it in grain and whatever, it's now in the AI and it thinks it's all legitimate because, well, it's in the documents. Right. Like that's an interesting kind of phenomenon where it's just reinforcing itself.

**Andrey Orlenko:** Yeah. This particular problem. Actually, I was thinking about this and okay, it. It's thought that you are a stakeholder. Right. Which is not what you want. My idea was. And I think it's still way to at least consider it. We have clients and for example, we implemented taxonomy at one point.

**Puru Agrawal:** Yes.

**Andrey Orlenko:** I was thinking about something like a client. Yes, A client should have a context. So basically in the system prompt we can say something like, Buru Agrawal is our cto. So he is part of Altus Nova. He is consultancy. Then I don't know. Jason is cfo. Right. And. Like we can list out all the folks who work for Altus Nova. Like he is a developer. This is like the team, actually the team only maybe for global Altus Nova. We outline only, let's say you. And then we have a client context where we provide. Okay, who is our favorite CSSI guy that we always ask about? Who is foreign? Desio. Right. So Warren Desio is this. And Andrea Orlenko is a tech lead. And so just give more context, client based context. With that alone, it's almost like a contacts page, right? Yes.

**Puru Agrawal:** And the contacts you can assign to roles and you can tag as stakeholders and you can tag as subject matter experts and maybe Anna has to take that into consideration or is capable of taking that into consideration.

**Andrey Orlenko:** Yes. Then on each query Anna will have that context and he will understand that. Okay, so Pura Graval is not stakeholder, so it would not give you that answer as before just because it already knows it.

**Puru Agrawal:** So there might be kind of an overall old definition, right. Where you say, okay, consultant. This is the description and what they do, the client, stakeholder, client, executive, blah blah blah. And as you define these roles here, then for a given client you might have a contacts page and it lists out all the contacts that it's seeing come up in any of the documents and stuff, right? Yes. And you can maybe tell it different contacts that are actually the same person. And so, so and so shows up like this in teams, but like this in Grain and like this in Slack and blah blah blah. And. But they're really all the same person, right? Yes.

**Alex Nanai:** And then I think that Andre actually meant more context in general. Like we have this Claude MD file which is included into each conversation. It's almost the same can be created. Right, but it's the same.

**Andrey Orlenko:** Yes. From what describes, you've got to build some way for

**Puru Agrawal:** a human to describe it and then Anna to pull it in, right?

**Andrey Orlenko:** Yes, yes.

**Petro Darii:** So also we can have multiple and multiple entities of same client. Yeah. I can have one client for CSSI and I have second client for ssi. The thing should be shared probably. Yeah.

**Puru Agrawal:** Just like taxonomy is shared. Right. The contacts and roles would be shared context. And so once you define it, it's going to pull it in for everybody who's in that client. Yeah.

**Andrey Orlenko:** Taxonomy is basically just a nice UI that still translates everything to text. What Peru outlined is user friendly way to actually generate then the text. Because all the LLM cares is text ultimately.

**Petro Darii:** Yeah.

**Puru Agrawal:** But probably Anna could search through the documents and suggest all the contacts who should be listed out in the contacts page, right?

**Andrey Orlenko:** Yes, yes, yes. Because we have these people everywhere. We know who the speaker is on the grain. We know the author of page in Confluence. Let's say we know Slack for author for each Slack message. So we have that context. We, we have people emails, we have their names. So.

**Alex Nanai:** In grain recordings it's also when person is speaking and saying I or me it always references person by their email and then when you say someone else it will write the name. But it can't really understand that this. So if I say I am going to implement it and Peru is saying Alex is going to implement it to Grain, it's two, it's two different people. Right. So I mean but here Anna is just like Andre said, it's going to. So it's important if it knows that this Slack person is actually this person. Right. So this Slack login is actually this person.

**Puru Agrawal:** Right. And I think it has that information. Right. So if you look at the grain transcripts like it has names associated with each statement. Yeah.

**Alex Nanai:** But if you, if someone, if, if Peru you are going to say Charlie is going to do something it will not so probably it will not map. At least I have seen those cases. It, it's not mapping. I think so anyway. So I was just going to say it's probably separate effort to make it to understand that this Slack login is actually Warren Desi or something. Yes.

**Puru Agrawal:** And that's why I was saying like there might be multiple IDs for the same contact. Right. And on that contact page we need some way of saying that this Slack ID and this Grain ID and this team's ID and this, whatever. Are all related to this one contact.