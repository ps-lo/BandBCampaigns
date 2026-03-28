**Scenario Name**: A random teams message
## Author: Steven Lorenz  
Insecure defaults, really?  
This is a "starter" style scenario. It plays through, but you will need to adlib some additional details throughout the game.  

## About Backdoors & Breaches
Backdoors & Breaches is an incident response card game created by the Black Hills Information Security Tribe of Companies https://www.blackhillsinfosec.com/tools/backdoorsandbreaches/  Purchase Backdoors & Breaches cards at https://spearphish-general-store.myshopify.com/collections/backdoors-breaches-incident-response-card-game or play online at https://play.backdoorsandbreaches.com  A big thank you to the team that created this great game!

## Game Setup  
**Decks** Core + Cloud (Physical or at play.backdoorsandbreaches.com)  

## Scenario Cards
Initial Compromise: Out-of-Band Phish
Pivot & Escalate: Malicious File Upload to Shared File Service 
C2 & Exfil: Cloud-based services as Exfil 
Persistence: MFA Bypass: App Password Creation

## Established Procedures (choose 4) 
Select any combination of 4 Procedure cards to be established procedures. I like to select 2-3 established procedures that can solve more than one scenario cards. The remainder can solve one or none 🧌. 

| Procedure | Initial Compromise | Pivot & Escalate | C2 & Exfil | Persistence |
| --- | --- | --- | --- | --- |
| SIEM Log Analysis | X | X | X | |
| Endpoint Security Protection Analysis | X | X | | |
| User and Entity Behavior Analytics | X |  |  | X |
| Endpoint Analysis |  | X | X |  |
| Memory Analysis |  | X |  |  |
| Network Threat Hunting |  |  | X |  | 
| Cloud Event Log Analysis |  |  |  | X |

## Scenario Brief
Who is involved? Sales Manager, Tim

Scenario (Read to responders)  
Tim: Hmm, what am I going to have for lunch today?  
**DING** A Teams Message from the boss. This must be important since Jim normally emails me.  
"Please look at this updated sales commission spreadsheet for your team."  
Tim: Thinking nothing of it, Tim opens the file and logs in to SharePoint. A blank workbook opens.
Tim responds to the teams message letting his boss know that the file was a blank workbook.
Boss: I'll resend the file after lunch. 

Tim: Well, nothing to do here, I'm going to lunch.

At this time, ask the responders how they would figure out if something bad happened and ask them if there are any questions.

## Playthrough
During gameplay, if the reponders roll a 1 - 10, have some reasons why the procedure fails. You will also need to do this if the responders choose a procedure that doesn't solve a card and they roll an 11 -20. These reasons can be office politics, finanicial, personnel based, etc.   

Some of my favorites:  
- The specialist that corresponds with the procedure is out to lunch, had bad gas station sushi last night, etc.  
- Finance forgot to pay the bill, the Splunk bill was larget than your credit cards limit, Management declined the UEBA purchase  
- In cases where an 11 - 20 is rolled but doesn't flip a card "Just because you are good at something doesn't mean it always works out"  

## Card details
As your responders flip cards, you should add additional details to the story  

- Initial Compromise- Out-of-band phish: Did you know, by default anyone can send a message to yout M365 tenant? Insecure defaults may have cost you.
- Pivot & Escalate- Malicious File Upload to Shared File Service: The user saw legitimate branding and logged in. At least they accessed a Excel file on something that looks like SharePoint. (Yes, this isn't a perfect fit, sometimes you need to use your imagination)  
- C2 & Exfil- Cloud-based services as Exfil: Once the adversary was in, they dumped all of files Tim had access to and saved them to mega.io
- Persistence- MFA Bypass: App Password Creation: Another insecure default. Users can create app passwords that bypass passwords and MFA on return trips to the Microsoft 365 tenant. 
