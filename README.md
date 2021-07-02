# Zabbix IT service tree

PostgreSQL query/function to retrieve Zabbix IT Services from service tree, starting from selected service (by it's name).

Function made especially to work with Grafana, but can be used also in other tools - I'm using it also in Jasper Reports.

## Purpose

Let's say we have Service tree in Zabbix like below:

    ─ Service A
        ├─ Service B
        └─ Service C
             ├─ Service D
             └─ Service E

Zabbix API "service.get" method allows to retrieve services according to the given parameters.

Problems:
1. If we ask for "Service A" we will get info that this service has child services "Service B" and "Service C", but response do not show us names but IDs of this services. So we need to run one more request to resolve names
2. If we also need child services to this services from point 1 (childs of Service B and C) then we need to run another request. If there will be more levels in tree then we need more requests
3. If there are more levels in the tree, we need to use more queries (recursion)

Function alows to:
1. Ask for services child by parent service name
2. Function return service names
2. We can choose how deep we want to search services child (how many levels to go into)

## How to use

To get names of child services to "Service A" run query:
```sql
SELECT * FROM its_servicetree_byname(1, 'Service A');
```
Response: "Service B" "Service C"

To get names of child services to "Service A" and child services to this services (level 2 in tree) run query:
```sql
SELECT * FROM its_servicetree_byname(2, 'Service A');
```
Response: "Service B" "Service C" "Service D" "Service E"

## Important note

Using service name is convenient, but Zabbix DB allows service names to be not unique (IDs are unique). So if you want to use this function you should use only unique services names, or modify this function to not throw error when it find duplicate names. If you are using not unique service names then you can't distinguish services by it's names.

## Code of the function

```sql
CREATE OR REPLACE FUNCTION public.its_servicetree_byname(tree_depth integer, servicename character varying)
 RETURNS TABLE(serviceupid bigint, parentname character varying, servicedownid bigint, childname character varying, path bigint[], depth integer)
 LANGUAGE plpgsql
AS $function$
DECLARE
	my_serviceid INT8;
BEGIN
SELECT services.serviceid INTO STRICT my_serviceid FROM services WHERE services.name = $2; 

RETURN QUERY 
WITH RECURSIVE nodes(serviceupid, parentName, servicedownid, childName, path, depth) AS (
SELECT
             r."serviceupid", p1."name",
             r."servicedownid", p2."name",
             ARRAY[r."serviceupid"], 1
       FROM "services_links" AS r, "services" AS p1, "services" AS p2
       WHERE r."serviceupid" = (SELECT services.serviceid FROM services WHERE services."name" = $2)
       AND p1."serviceid" = r."serviceupid" AND p2."serviceid" = r."servicedownid"
       UNION ALL
       SELECT
             r."serviceupid", p1."name",
             r."servicedownid", p2."name",
             nd.path || r."serviceupid", nd.depth + 1
       FROM "services_links" AS r, "services" AS p1, "services" AS p2,
             nodes AS nd
       WHERE r."serviceupid" = nd.servicedownid
       AND p1."serviceid" = r."serviceupid" AND p2."serviceid" = r."servicedownid" AND nd.depth < $1
)
SELECT * FROM nodes;

EXCEPTION
	WHEN NO_DATA_FOUND THEN
		RAISE EXCEPTION 'service: % not found', $2;
	WHEN TOO_MANY_ROWS THEN
		RAISE EXCEPTION 'service: % not unique', $2;

END
$function$
;
```

> Tested on PostgreSQL 12.x and Zabbix 5.0.x
