---
layout: post
title: "My first OpenBSD Kernel Bug"
date: 2007-07-05 19:42
comments: true
categories: ramblings
---

Mit `pf`, `pfsync` und [CARP](http://en.wikipedia.org/wiki/Common_Address_Redundancy_Protocol "Wikipedia: Common Address Redundancy Protocol") bietet OpenBSD eine vergleichsweise einfach zu administrierende und dennoch leistungsfähige Infrastruktur für das Failover redundant aufgesetzter Firewalls. Neben dem essentiellen Paketfilter `pf` kümmert sich `pfsync` in diesem Szenario um das synchrone Statehandling der verschiedenen Nodes, was bei einem Schwenk des aktiven Firewallrechners im Cluster bestehende Netzwerkverbindungen aufrecht erhält. Der Einsatz des Common Address Redundancy Protocols CARP erledigt auf OSI-Schicht 2 und 3 dabei das eigentliche Procedere zur Hochverfügbarkeit, bei dem es klassischerweise zwischen MASTER- und BACKUP-Rollen auf Interface-Ebene unterscheidet. Eine trotz ihrer Kürze detailreich geschriebene Übersicht zu diesem Thema bietet der Artikel [“Firewall Failover with pfsync and CARP”](http://www.countersiege.com/doc/pfsync-carp/).

Das lange überfällige Upgrade eines schon etwas in die Jahre gekommenen OpenBSD 3.7 Clusters auf Release 4.1 gab mir die Gelegenheit, meinen ersten OpenBSD-Kernelbug - wenn auch eher unfreiwillig und in gewohnt unpassendster Situation - zu entdecken: ein sequentielles Neuladen der Regeln des auf zwei Nodes verteilten Paketfilters führte nach wenigen Iterationen regelmäßig zum Absturz des Systems:

{% codeblock %}
kernel: page fault trap, code = 0
Stopped at	pfsync_insert_net_state+0x472:	movl	0(%eax,%edx,4),%edx
{% endcodeblock %}

Mit der Analyse des Trace-Outputs (im OpenBSD-Kerneldebugger `ddb` per `trace` aufgerufen) kann die betreffende Funktion erkannt und im Quellcode schnell lokalisiert werden. Im Falle des gestorbenen Firewallnodes lässt folgende Ausgabe das Problem auf das Sourcefile `sys/net/if_pfsync.c` zurückführen:

{% codeblock %}
pfsync_insert_net_state(e34d4038,1,8,e34d4038) at pfsync_insert_net_state+0x472

pfsync_input(e3486a00,14,0,0,d0d1a034) at pfsync_input+0xa21
ipv4_input(e3486a00,d0d0e900,0,d08ab000,30) at ipv4_input+0x4f1
ipintr(d0640058,d30010,d08a0010,d08a0010,d08ab000) at ipintr+0x70
Bad frame pointer: 0xd08ace24
{% endcodeblock %}

Erneut mit Debuginformationen kompiliert und disassembliert finden wir per `grep` die fehlerhafte Funktion und addieren der dort angegebenen Speicheradresse den Offset aus unserem Trace-Output hinzu:

{% codeblock %}
# grep “<pfsync_insert_net_state>” if_pfsync.dis
> 000002f4 <pfsync_insert_net_state>
{% endcodeblock %}

Adam Riese addiert die Hexadezimalzahlen `0x2f4` + `0x472` zu `0x766` - genau in jener Zeile sollte sich innerhalb unseres disassemblierten Codes die Instruktion aus unserem Kerneltrap finden, und siehe da:

{% codeblock %}
/usr/src/sys/net/if_pfsync.c:248
      756:       a1 b4 04 00 00          mov    0x4b4,%eax
      757:       R_386_32   pf_main_anchor
      75b:       66 c1 ca 08             ror    $0x8,%dx
      75f:       c1 ca 10                ror    $0x10,%edx
      762:       66 c1 ca 08             ror    $0x8,%dx
      766:       8b 14 90                mov    (%eax,%edx,4),%edx
      769:       89 55 ec                mov    %edx,0xffffffec(%ebp)
      76c:       e9 e5 fb ff ff          jmp    356 <pfsync_insert_net_state+0x62>
      771:       8d 76 00                lea    0x0(%esi),%esi
{% endcodeblock %}      

Wir haben damit die genaue Zeilenangabe des betreffenden Codeteils innerhalb der Funktion `pfsync_insert_net_state` gewonnen, welche wir schon im Sourcefile `sys/net/if_pfsync.c` festmachen konnten. Mit etwas Kontext sprechen also alle Indizien das folgende Konstrukt schuldig:

{% codeblock if_pfsync.c %}
/*
* If the ruleset checksums match, it’s safe to associate the state
* with the rule of that number.
*/
if (sp->rule != htonl(-1) && sp->anchor == htonl(-1) && chksum_flag)
        r = pf_main_ruleset.rules[
            PF_RULESET_FILTER].active.ptr_array[ntohl(sp->rule)];
else
        r = &pf_default_rule;
{% endcodeblock %}

Tiefer bewanderte Kerneldeveloper identifizieren hier eine Racecondition zwischen den Ruleset-Reloads beider Maschinen und stellen - keine 24 Stunden nach meinem Bugreport - den ersten Patch zur Evaluation, der mittlerweile mit Revision 1.83 im CVS des MAIN-Branches enthalten ist. Ich konnte ihn erfolgreich ohne weitere Abstürze unter Stress setzen:

{% codeblock if_pfsync.c %}
/*
* If the ruleset checksums match, it’s safe to associate the state
* with the rule of that number.
*/
if (sp->rule != htonl(-1) && sp->anchor == htonl(-1) && chksum_flag &&
    ntohl(sp->rule) <
    pf_main_ruleset.rules[PF_RULESET_FILTER].active.rcount)
        r = pf_main_ruleset.rules[
            PF_RULESET_FILTER].active.ptr_array[ntohl(sp->rule)];
else
        r = &pf_default_rule;
{% endcodeblock %}        

Das Fazit: Selbst durchaus unerfreuliche Vorkommnisse bieten bei Verfügbarkeit des Quellcodes das Potential, ungekannte Hintergründe zu verstehen und dabei stets neues zu erlernen. Die Kommunikation mit den Entwicklern freier Software lässt darüber hinaus das gute Gefühl entstehen, selbst als einfacher Bote einer schlechten Nachricht bei der stetigen Verbesserung der quelloffenen Produkte mitwirken zu können. 

Denn auch davon lebt Open Source: den ausführlichen Bugreports engagierter User.