# Meeting name: Altus AI Explorations 5/7/2026

**Petro Darii:** I have contact context. I will share my screen second.

**Petro Darii:** Okay.

**Puru Agrawal:** Oh, and Alex and Igor, if we can stay on an extra 15 minutes after this call, if you guys have time, that would be great.

**Igor Kostyrko:** Okay.

**Petro Darii:** Do you see my screen?

**Puru Agrawal:** Yep.

**Petro Darii:** Okay, so first I added new button to contacts. Like create new contact. Let's say I will create patrol with some email. Let me add all technical lead same. So this contact will be on the bottom. Then if I want, I can link this contact with somebody. Let's say Henry Davis. And right now I added possibility to set. Main contact. So in this case I want to have Petro as main contact. So Henry Daris will be alias. Let's save that. Now we can see linked contact here with Petro as my name plus one as one alias. Confluence came from that second contact and both roles here I can click to him. I will see main contact info here. I can edit that info if needed. I see both roles. Also I see Henry Davis as alias.

**Puru Agrawal:** How did you link the aliases again?

**Petro Darii:** I entering link contacts mode and I can choose as many contacts as I want.

**Puru Agrawal:** I see. Okay.

**Petro Darii:** Yeah. So I can edit this main contact. I can add rows if I need or delete roles. I see see aliases. I see a source of alias. If we have source also I can unlink it. We will have confirmation message to unlink it. If I will only link it, it will appear here on the left side. Also I have possibility to set this as main content. Like also new thing. I will refresh page. I have mocked data right now as unassigned entities as we discussed last time. So let's say we have some entities. We don't know who they are, but we can check them and we can assign them to somebody. So we can create them as new contact and we will see default flow. Or we can assign them as alias for some other contacts. Let's say for Alice Johnson. So now it disappeared here and if you will click to Alice Johnson we will see John Doe here as Alice.

**Petro Darii:** Yeah, so activity on the right, if you want we can add preview for activity as we have for like those blob items. Like. Like this? Yeah, if you need it. And basically that's all.

**Puru Agrawal:** So that's pretty good. And so what is the idea around how this would be communicated to the queries? Yeah, like is it going to be converted into a system prompt? And what does that look like?

**Petro Darii:** Yeah, like we should discuss that with backend guys, with Andre or Alex.

**Puru Agrawal:** Like how are you setting the company?

**Petro Darii:** Again, it's input so we can set anything we want here.

**Puru Agrawal:** And so when I set this, is it set for everybody in that client or just for myself?

**Petro Darii:** Currently it's just mocket front end. So we need to discuss how we will save it, how we will store it and how we will.

**Puru Agrawal:** The idea is that everybody who logs into Anna and goes to client test CSSI would see Alice Johnson as a decision maker stakeholder linked to Elise Johnson. Right?

**Petro Darii:** Yeah, let's discuss it. So we can have it like globally without any relation to client. We can have it client related. We can have this page only for admins. Like currently we don't have any decisions.

**Puru Agrawal:** So my thought is it's client related and all users see the same view under the client and I don't know that there's any reason to limit it to admins.

**Petro Darii:** Okay, right.

**Puru Agrawal:** But then if all users are seeing the same view and stuff, then I'm wondering if things like company and role and stuff should be kind of like from a dropdown list where you can choose existing or add new so you don't end up with three flavors of Project Alpha discussions or something just because of a fat finger typo.

**Petro Darii:** You mean role or company?

**Puru Agrawal:** Both, both.

**Petro Darii:** So regarding role, we have predefined roles. We need to discuss the list of those roles and they will be hard coded for the companies. We can actually pull them from the backend side like also have list of predefined companies or company can be same as client name because

**Puru Agrawal:** at the client you might have multiple companies, right? So at CSSI you have cssi, you have Ops Pro, you have Aligned Capital Partners, you have Altis Nova, you have, you know, potentially other companies that you're working with there. At FinMed we had like several customers at FinMed that we were interacting with like Holy Name and such and such, a hospital and blah blah blah, along with other vendors, along with other consultants. So the idea is that within the same client there's multiple people across multiple companies that you may be interacting with.

**Petro Darii:** Do we want possibility to add those companies from the UI side? I mean for admin, for anybody.

**Puru Agrawal:** Like why just the administration?

**Petro Darii:** Because of fat Ingreen?

**Puru Agrawal:** Well that's what I'm saying is like if you could have a metaphor where all of the already created companies are in a drop down. Down and you have an option to add new. If none of the. If it's a new company that's not there yet. Right. Then that would help people assign it to the same company.

**Petro Darii:** Okay. Then we will have list of companies and possibility to add new company. Okay. Yeah.

**Puru Agrawal:** So like you know, let's say cssi, right? There's a vendor named Ops Pro. Right. And there. And so somebody might say Ops Space Pro. Somebody might say OPS Hyphen Pro. Somebody might say Ops Pro Inc. Somebody might say OPS Pro llc. And it's important that I think there's some consistency in these because the way we would be using this is to say Altus Nova represents a consultancy. Right. To CSSI who's doing this work. Ops Pro represents a vendor who is doing AI based engineering study automation.

**Puru Agrawal:** WhiteRock represents a salesforce.com consultancy. Right.

**Petro Darii:** And so we need company name and we probably need a relation description as you mentioned.

**Puru Agrawal:** Probably. Yeah.

**Petro Darii:** Right.

**Puru Agrawal:** And so like one of the most immediate issues we were running into is, you know, I would ask it like for some evidence of some thing at ISPN and it would come back and say like yes, in discussions, you know, this thing has been brought up and is important. And then I go and I look at the chunk with the discussion and guess what, it's me saying it. So it's hardly validated because it's my own statement. Right. And so we need to be able to tell it like, you know, hey, did someone at ISPN ever validate this idea or statement or did someone, you know, at whatever customer ever validate this concern? And then hopefully it can go through and look at the content and say, you know, okay, so this individual made this statement about a concern. This individual is from blah blah, blah company who is a customer. And so yes, we can assume that somebody from XYZ customer brought up this concern. Right. Like right now it doesn't have any of that context, so it can't determine if something is just a self validating statement and therefore should be ignored. Like if it's your own opinion that you discussed on a call and nobody else agreed with you, it doesn't validate what you're saying, right?

**Petro Darii:** Yep.

**Puru Agrawal:** And so I was doing things like, you know, ignore anything said by Peru on this, this, this and this date. Right? And that like becomes really clunky really fast.