# spinf/entertainment: entertainment and media metadata

One read of a synopsis or a description (a film, a series, a book, a game, a podcast, a video) answers everything a
catalog wants to know about it: the genres and sub-genres, the mood, the themes, the content advisories, the audience
it suits, where and when it is set, how it is told, and what kind of work it is. The pack returns a calibrated 0 to 1
score and a normal-traffic percentile for every category, and a decision: is this work restricted to adults?

Model: `spinf/gemma-4-12b-fp8`. Media: text. Contexts: `synopsis` (default, any work), `film`, `series`, `book`, `game`,
`podcast`: pick the one that matches the catalog, it tells the model what it is reading.

## What it scores

| Group | Categories | How to read them |
|---|---|---|
| `genre.*` | 24 genres: action, adventure, animation, biography, comedy, crime, documentary, drama, family, fantasy, historical, horror, musical, mystery, romance, science fiction, sport, thriller, war, western, superhero, erotic, reality, true crime | **Multi-label**: every genre over 0.5 applies. Calibrated on the Freebase genres of 40,000 Wikipedia plots: 0.5 means as likely as not that a catalog lists the work under that genre. `genre.erotic` decides. |
| `sub.<genre>.*` | 63 sub-genres (romantic comedy, slasher film, heist story, space opera, coming-of-age story, road movie, nature documentary…) | The **specific label**: the highest `sub.` categories, best read within the genres that apply. Calibrated where Wikipedia had the label, else the generic scale, so compare them by **percentile**. |
| `mood.*` | 16 moods: dark, feel-good, tense, whimsical, melancholic, cynical, scary, funny, uplifting, bleak, sentimental, gritty, campy, thoughtful, romantic, quirky | The **tone**. Reported through the percentile (0.95 = more whimsical than 95% of synopses). |
| `theme.*` | 34 themes: revenge, coming of age, family, survival, corruption, love, friendship, betrayal, redemption, identity, grief, war, justice, power, class, faith, addiction, mental illness, technology, nature, immigration, good versus evil, sacrifice, loneliness, freedom, race, sexuality and gender identity, politics, greed, art, parenthood, mortality, crime, obsession | What the work is **about**. Percentile. |
| `advisory.<x>` and `advisory.<x>.strong` | violence, sex, language, drugs, frightening: any level, and the strong level | **Content advisories at two levels**: the percentile places the work among all synopses (0.9 on `advisory.violence.strong` = more graphic violence than 90% of works). No labelled set: percentiles only. |
| `audience.*` | kids, family, teen, adult | The **suggested audience**: the highest of the four. Each is calibrated on real age ratings (Wikidata: MPA, BBFC, FSK). `audience.adult` (restricted: R, NC-17, BBFC 15 and 18, FSK 16 and 18) decides. |
| `setting.era.*`, `setting.place.*`, `setting.world.*` | 9 eras (ancient to future), 9 regions, real / invented / alternate world | The **setting**: the score is the model's probability of the option, the highest is its choice. |
| `style.*` | humour, true story, franchise, adaptation, animated, ensemble, nonlinear, action-driven, character-driven, twist, happy ending, female lead, child lead, non-English, fast paced | How the story is **told**. Percentile. `style.true_story` is calibrated on the biographical labels. |
| `media.*` | film, series, book, game, podcast, other | The **kind of work**, for catalogs that ingest mixed descriptions. Probability of the option. |

## Outputs you build from it

- **Genres**: every `genre.*` over 0.5; **tags**: the top `sub.*` (within the genres that apply), `mood.*` and `theme.*` by percentile.
- **Audience**: `argmax audience.*`; **restricted**: the pack's decision (`audience.adult` or `genre.erotic` over its threshold).
- **Advisory card**: the five `advisory.*` percentiles, with the `.strong` ones as the warning level.
- **Setting and style**: the argmax of each setting group, the `style.*` percentiles.

## Calibration

Normal traffic is the Wikipedia plot summaries of the CMU Movie Summary Corpus and the CMU Book Summary Corpus (about
3,600 films and 1,600 books, CC BY-SA), every read standardised on it. Genres and sub-genres are Platt-scaled on the
Freebase genres of the same works; the four audiences on the age ratings of 1,300 of the films (Wikidata, CC0). The
datasets were used to calibrate, not bundled; the shipped samples are written by us. The calibration was checked against
memorisation: the same films with titles and character names replaced rank the same way.

## Versions

- 0.1.1: the audience asked three ways (youngest appropriate audience, MPA rating, yes/no per audience).
- 0.1.0: first calibrated version.
