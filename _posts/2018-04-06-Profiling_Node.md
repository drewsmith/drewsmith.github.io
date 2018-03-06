---
title:  "Profiling Node"
categories: Web JS
---

Recently I ran into a memory leak while running a node script to migrate ~300MM rows from Postgres to DynamoDB.

The first step was to add [node-memwatch](https://github.com/lloyd/node-memwatch):

```
const memwatch = require('memwatch');

memwatch.on('leak', (info) => console.log("Possible leak: ", info));
```

This outputs an info object similar to:

```
{ start: Fri, 29 Jun 2012 14:12:13 GMT,
  end: Fri, 29 Jun 2012 14:12:33 GMT,
  growth: 67984,
  reason: 'heap growth over 5 consecutive GCs (20s) - 11.67 mb/hr' }
```

Unfortantely, this identifies issues that are more a symptom or warning than a cause.

## Node Inspector

Thankfully, Node has a built-in inspect argument:

*package.json*

```
"scripts": {
  "run": "node --inspect index.js"
}
```

Adding `--inspect` attaches the node debugger, which you can read more on here:

[https://nodejs.org/en/docs/inspector/](https://nodejs.org/en/docs/inspector/)

I found the easy way to profile a Node app is to use the [NiM Chrome Extenion](https://chrome.google.com/webstore/detail/nodejs-v8-inspector-manag/gnhhdgbaldcilmgcpfddgdbkhjohddkj?hl=en), which produces output similar to this:

![Node debugger]({{ "/assets/images/node_profiling.png" | absolute_url }})

If necessary, you can also bump up memory as well, but I would only do this if necessary and bumping up memory is not fixing the root cause:

```
"scripts": {
  "run": "node --inspect --max-old-space-size=2048 index.js"
}
```

# Bonus, async for loops

A tangent that I ran into while accomplishing this was asynch handling in for loops, which natively execute in synchronously. This is an example solution that I came up with:

```
const forEachAsync = async (array, callback) => {
  for (let index = 0; index < array.length; index++) {
    await callback(array[index], index, array);
  }
}

const UserService = {
  getName: async (user) => `${user.firstName} ${user.lastName}`
}

const users = [
  { firstName: 'Babe', lastName: 'Ruth' },
  { firstName: 'Jimi', lastName: 'Hendrix' },
  { firstName: 'George', lastName: 'Brett' },
  { firstName: 'Winston', lastName: 'Churchill' },
];

const getUserNames = async () => {
  await forEachAsync(users, async user => {
    try {
      let name = await UserService.getName(user);
      console.log("Name: ", name);
    } catch (err) {
      console.error(err.message);
    }
  });
  console.log("getUsers complete.");
};

(async () => {
  try {
    await getUserNames();
  } catch (err) {
    console.log(err.message);
  }
  console.log("Complete");
})();
```

Output:

```
Name:  Babe Ruth
Name:  Jimi Hendrix
Name:  George Brett
Name:  Winston Churchill
getUsers complete.
Complete
```
