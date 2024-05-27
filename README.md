[![FIWARE Banner](https://fiware.github.io/tutorials.Administrating-XACML/img/fiware.png)](https://www.fiware.org/developers)

[![FIWARE Security](https://nexus.lab.fiware.org/repository/raw/public/badges/chapters/security.svg)](https://github.com/FIWARE/catalogue/blob/master/security/README.md)
[![License: MIT](https://img.shields.io/github/license/fiware/tutorials.Administrating-XACML.svg)](https://opensource.org/licenses/MIT)
[![Support badge](https://img.shields.io/badge/tag-fiware-orange.svg?logo=stackoverflow)](https://stackoverflow.com/questions/tagged/fiware)
[![FIWARE Security](https://img.shields.io/badge/XACML-3.0-ff7059.svg)](https://docs.oasis-open.org/xacml/3.0/xacml-3.0-core-spec-os-en.html)

This tutorial describes the administration of level 3 advanced authorization rules into **Authzforce**, either directly,
or with the help of the **Keyrock** GUI. The simple verb-resource based permissions are amended to use XACML and new
XACML permissions added to the existing roles. The updated ruleset is automatically uploaded to **Authzforce** PDP, so
that policy execution points such as the **PEP proxy** are able to apply the latest ruleset.

The tutorial demonstrates examples of interactions using the **Keyrock** GUI, as well [cUrl](https://ec.haxx.se/)
commands used to access the REST APIs of **Keyrock** and **Authzforce** -
[Postman documentation](https://www.postman.com/downloads/) is also available.


# Start-Up

**NGSI-v2** offers JSON based interoperability used in individual Smart Systems. To run this tutorial with **NGSI-v2**, use the `NGSI-v2` branch.

```console
git clone https://github.com/FIWARE/tutorials.Administrating-XACML.git
cd tutorials.Administrating-XACML
git checkout NGSI-v2

./services create
./services start
```

| [![NGSI v2](https://img.shields.io/badge/NGSI-v2-5dc0cf.svg)](https://fiware-ges.github.io/orion/api/v2/stable/) | :books: [Documentation](https://github.com/FIWARE/tutorials.Administrating-XACML/tree/NGSI-v2) | <img  src="https://cdn.jsdelivr.net/npm/simple-icons@v3/icons/postman.svg" height="15" width="15"> [Postman Collection](https://fiware.github.io/tutorials.Administrating-XACML) |
| --- | --- | --- |

---

## License

[MIT](LICENSE) © 2019-2024 FIWARE Foundation e.V.
