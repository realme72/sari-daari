# sari-daari

> *Sari daari* (ಸರಿ ದಾರಿ): "the right way".

Bengaluru was the world's 2nd most congested city in 2025 (TomTom Traffic Index): rush-hour
speeds of about 14 km/h, and about 168 hours a year lost to traffic. Sari-Daari answers one question for each trip:

**Metro, bus, or my own vehicle, and which station or stop do I walk to?**

- Door-to-door comparison: metro, BMTC bus, or your own bike/car, including the walk or auto to the station or stop
- Nearby metro stations and bus stops, and the right one to walk to
- Live traffic on your route, plus alternate routes
- Peak-hour warnings and leave-time advice
- Saved commutes with alerts before you usually leave

Status: **design phase**. See [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md).

## Data & credits

- Metro: [Vonter/bmrcl-gtfs](https://github.com/Vonter/bmrcl-gtfs) (unofficial GTFS, ODbL 1.0), from timetables published by [BMRCL](https://www.bmrc.co.in)
- Bus: [Vonter/bmtc-gtfs](https://github.com/Vonter/bmtc-gtfs) (unofficial GTFS, ODbL 1.0), from the Namma BMTC app
- Traffic & routing: [TomTom](https://developer.tomtom.com)
- Map data © [OpenStreetMap](https://www.openstreetmap.org/copyright) contributors

## License

MIT for the code. Third-party data keeps its own licence (see above).
