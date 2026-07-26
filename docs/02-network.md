# 02 - Configuration Réseau

## Interfaces OPNsense

| Interface | Carte | Rôle | Adresse IP | Destination |
|-----------|-------|------|------------|-------------|
| **WAN** | igc3 | Sortie Internet | 192.168.5.244/24 | Réseau FAI |
| **LAN** | igc2 | Clients de confiance | 10.0.0.1/24 | AlphaDeck + machines locales |
| **OPT1** | igc1 | Zone Bastion | 10.0.10.1/24 | Prodesk |
| **OPT2** | igc0 | Zone isolée / Lab | 10.0.20.1/24 | GX10 |

## Adresses de la Prodesk (OPT1)

| Rôle | Adresse IP | Usage |
|------|------------|-------|
| **Services** | 10.0.10.10/24 | BunkerWeb / services web |
| **Management** | 10.0.10.11/24 | SSH / administration |

## Routing

- Route vers LAN depuis OPT1 : `10.0.0.0/24 via 10.0.10.1`

## Voir aussi

- [01 - Architecture](./01-architecture.md)
- [06 - Aliases OPNsense](./06%20-%20Aliases%20OPNsense.md)
- [04c - VLANs et Segmentation](./04c_VLANs%20et%20Segmentation%20Réseau.md)
