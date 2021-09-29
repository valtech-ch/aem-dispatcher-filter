# AEM Dispatcher Filter Best Practices

The AEM Dispatcher Filter best practice project was initially created together with Adobe's EMEA AEM practice to address shortcomings in the HelpX documentation, with the longer term aim of contribution to HelpX.

In general we find that dispatcher filters 'in the wild' are poorly implemented and as a result exploits that make use of crafted URLs to evade filtering are common.  Key problems are:

- The OOTB dispatcher filter files on which most rulesets are based are very weak. The archetype filters are designed primarily to ensure that AEM works when newly set-up and contribute little to security in their raw form.

- An accepted firewall best practice of deny/allow is only losely followed in most live dispatcher filter rulesets, that is, paraphrased:

```
Deny all
Allow almost all
Deny
Deny
Deny
etc
```

This is a fundamentally weak approach.

- Developers rarely apply simple clean-coding type norms to filter configuration.  

- The dispatcher filter is normally an afterthought in feature development

### Content

The project consists of an example dispatcher filter file and best practice documentation.

The example filter file is not suitable for direct copy'n'paste use as every client is different, however most rules can be quickly copied and adapted.  

## Security is ever more important - let's get one of the the most important AEM security features right
## Please contribute!
