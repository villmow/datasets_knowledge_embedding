# Task

I wanted to work with the datasets WN18 and WN18RR that contain 18/11 relations from wordnet data.

The original datasets have the following form:
```
02174461  _hypernym 02176268
05074057  _derivationally_related_form  02310895
08390511  _synset_domain_topic_of 08199025
02045024  _member_meronym 02046321
01257145 _derivationally_related_form 07488875
...
```
I wanted to have the textual form of the entities. Only the wordnet offsets 
are given as entites, transforming them back is problematic cause they are 
ambiguous within the 4 datafiles from wordnet. 

For example `01257145 _derivationally_related_form 07488875` has two offsets:
`01257145` and `07488875`. 

|      | **01257145**    | **07488875**     |
|------|-----------------|------------------|
| ADJ  |`sensual.s.02`   |                  |
| ADV  |                 |                  |
| NOUN |`precession.n.02`|`sensuality.n.01` |
| VERB |                 |                  |

I transformed the dataset back to wordnet synsets by validating if the given
relation holds on the ambiguous entities. 

The transformed textual data then looks like this:

```
clangor.v.01  _hypernym sound.v.02
straightness.n.02 _derivationally_related_form  straight.a.02
militia.n.01  _synset_domain_topic_of military.n.01
alcidae.n.01  _member_meronym pinguinus.n.01
sensual.s.02  _derivationally_related_form  sensuality.n.01
```

You can load it into NLTK by executing

```python
from nltk.corpus import wordnet as wn
wn.synset('sensual.s.02')
```
