> [!card] Nmap — Network Scanning
> Full scan with service detection
>
> > [!code-card] CLASSIC SCAN
> > ```bash
> > nmap -sC -sV -oA scan <TARGET>
> > ```
>
> > [!code-card] FULL PORT SCAN
> > ```bash
> > nmap -p- -T4 --min-rate=1000 -oN full_scan <TARGET>
> > ```
>
> > [!code-card] UDP SCAN
> > ```bash
> > sudo nmap -sU --top-ports 50 -oN udp_scan <TARGET>
> > ```