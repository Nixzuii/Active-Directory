# Introduction to Active Directory.

Picture I am administering a small business network with only 5 computers and five employees. In such a tiny network, I will probably be able to configure each computer separately without a problem. I can manually log into each computer, create users for whoever will use them, and make specific configurations for each employee's accounts. If a user's computer stops working, you will probably go to their place and fix the computer on-site.  

This sounds like a very relaxed lifestyle. Lets suppose my business suddenly grows and now has 150 computers and 300 different users located across four different offices. Would I still be able to manage each computer as a separate entity, manually configure policies for each of the users across the network and provide on-site support for everyone? The answer is most likely no.  

To overcome these limitations, we can use a Windows domain. Simply put, a Windows domain is a group of users and computers under the administration of a given business. The main idea behind a domain is to centralise the administration of common components of a Windows computer network in a single repository called Active Directory (AD). The server that runs the Active Directory services is known as a Domain Controller (DC).

<img width="452" height="362" alt="image" src="https://github.com/user-attachments/assets/010c017a-6f4e-477c-a1cd-37bc292a8009" />

The main advantages of having a configured Windows domain are:

- **Centralised identity management**: All users across the network can be configured from Active Directory with minimum effort.
- **Managing security policies**: We can configure security policies directly from Active Directory and apply them to users and computers across the network as needed.
