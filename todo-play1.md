Play : la TODO list
=================================

Nous allons réaliser aujourd'hui une simple `TODO list` en utilisant diverses APIs misent à disposition par Play ainsi qu'une bonne dose de Javascript. L'application à réaliser ressemblera à [quelque chose comme ça](http://ancelin.org/example/). Cette application exposera des services REST qui seront consommés par une IHM en Javascript.

Commencez par télécharger le squelette du TP [ici](https://raw.githubusercontent.com/mathieuancelin/poitiers-2015/master/todo-play1-starter.zip). 
Le projet contient déjà le minimum vital pour avoir une application qui fonctionne. De plus le projet contient une IHM de démo pour tester vos services ainsi qu'un outil de diagnostic pour vérifier que votre implémentation tient la route.

I. Partie Serveur
--------------------------

Premièrement, nous allons créer la couche de données de l'application. Pour celà créer une classe models.Task.java

```java
package models;

import play.db.jpa.Model;

import javax.persistence.Entity;

@Entity
public class Task extends Model {

    public String name;
    public Boolean done;
}
```

Maintenant, il faut créer les services REST correspondant dans la class Application.java

Cette classe doit répondre au contrat suivant :

* GET    /api/tasks        => récupération de toutes les instances de tâches
* POST   /api/tasks        => création une nouvelle tâche 
* PUT    /api/tasks/{id}   => mise à jour d'une tâche
* DELETE /api/tasks        => suppression de toutes les tâches
* DELETE /api/tasks/done   => suppression des tâches finies
* GET    /api/tasks/{id}   => récupération d'une tâche
* DELETE /api/tasks/{id}   => suppression d'une tâche en particulier

Voici un exemple de classe répondant à ce contrat

```java
package controllers;

import play.*;
import play.mvc.*;

import java.util.*;

import models.*;

public class Application extends Controller {

    public static void index() {
        render();
    }

    public static void allTasks() {
        // A implémenter
    }

    public static void createTask(String name) {
        // A implémenter
    }

    public static void deleteAll() {
        // A implémenter
    }

    public static void deleteAllDone() {
        // A implémenter
    }

    public static void getTask(Long id) {
        // A implémenter
    }

    public static void changeTaskState(Long id, Boolean done) {
        // A implémenter
    }

    public static void deleteTask(Long id) {
        // A implémenter
    }

}

```

Nous pouvons maintenant tester nos services. Pour celà, utilisez `curl` ou un plugin navigateur type `POSTman`


```sh
$ curl -X GET http://localhost:9000/api/tasks
[]
```

```sh
$ curl -X POST -d 'name=Hello' http://localhost:9000/api/tasks
{"name":"Hello", "id":1, "done":false}
$ curl -X POST -d 'name=Hello2' http://localhost:9000/api/tasks
{"name":"Hello2", "id":2, "done":false}
```

```sh
$ curl -X GET http://localhost:9000/api/tasks
[{"name":"Hello", "id":1, "done":false}, {"name":"Hello2", "id":2, "done":false}]
```

```sh
$ curl -X PUT -d 'done=true' http://localhost:9000/api/tasks/1
{"name":"Hello", "id":1, "done":true}
```

```sh
$ curl -X GET http://localhost:9000/api/tasks
[{"name":"Hello", "id":1, "done":true}, {"name":"Hello2", "id":2, "done":false}]
```

```sh
$ curl -X DELETE http://localhost:9000/api/tasks/done
```

```sh
$ curl -X GET http://localhost:9000/api/tasks
[{"name":"Hello2", "id":2, "done":false}]
```

```sh
$ curl -X DELETE http://localhost:9000/api/tasks
```

```sh
$ curl -X GET http://localhost:9000/api/tasks
[]
```

Nous pouvons maintenant passer à la partie client

II. Partie Client
---------------------------

Nous allons coder la partie client en Javascript en utilisant JQuery. Vous pouvez utiliser n'importe quel autre framework JS du moment que vous savez l'utiliser. Je vous conseille cependant de regarder du côté d'Angular et de React. 

Vous devrez écrire votre code javascript dans `web/js/todo.js`. Commencez par effacer les lignes 

```javascript
var el = ...;
$('#todo').html(el);
```

qui affichent le message `Votre application ici ;-)` et cachent un template de base pour votre application. La partie HTML de l'application est disponible dans `web/index.html`.

Voici le template html de l'application, ou vous pouvez aller consulter  :

```html
<div class="col-md-12">
  <h3>Todo List</h3>
  <div>
    <div class="row">
      <form role="form">
        <div class="form-group col-md-10">
          <input id="name" placeholder="What do you have to do ?" type="text" class="form-control">
        </div>
        <div class="form-group">
          <div class="btn-group">
            <button id="save" type="button" class="btn btn-success">
              <span class="glyphicon glyphicon-floppy-saved"></span>
            </button>
            <button id="delete" type="button" class="btn btn-danger">
              <span class="glyphicon glyphicon-trash"></span>
            </button>
          </div>
        </div>
      </form>
    </div>
  </div>
  <ul id="tasks" class="list-group">
    <!-- Ici itérer sur le tableau de tâche du modèle -->
    <li class="list-group-item">
      <div class="row">
        <div class="col-md-10">Hello</div>
        <div class="col-md-2">
          <span class="label label-success" style="cursor: pointer;">Done</span>
        </div>
      </div>
    </li>
    <!-- -->
  </ul>
</div>
```

Les différents cas à couvrir pas votre IHM sont les suivant :

* Lors du clic sur le bouton 'Add', si le texte entré dans l'input text du formulaire n'est pas vide, créer une tâche (via le service REST approprié) et l'ajouter dans la liste de tâche.

```html
<li class="list-group-item">
    <div class="row">
        <div class="col-md-10">
            <span>Nom de la tache</span>
        </div>
        <div class="col-md-2">
            <span class="label label-success">Done</span>
        </div>
    </div>
</li>
```

* Lors du clic sur une tâche, marquer la tâche comme effectuée (via le service REST approprié) et appliquer le style `label-success` sur le `span` de la tâche afin qu'elle apparaisse barrée.
* Lors du clic sur le bouton remove, toutes les tâche marquées comme effectuée sont effacées (via le service REST approprié) et sont retirées de la liste de tâches.

Pour pouvoir finir la vue, voici quelques snippets :

* Pour faire un get http :

```javascript
$.get('url', function(data) {
    // data contient les données retournées par les services REST
});
```

* Pour faire un post http :

```javascript
$.post('url', {param1:val1, param2:val2}, function(data) {
    // data contient les données retournées par les services REST
});
```

* Pour faire un put http :

```javascript
$.ajax({ url: 'url', type: 'put', data: {param1: val1}, success: function(data) {
    // data contient les données retournées par les services REST
}});
```

* Pour faire un delete http :

```javascript
$.ajax({ url: 'url', type: 'delete', data: {param1: val1}, success: function(data) {
    // data contient les données retournées par les services REST
}});
```

* Pour définir une action lorsqu'on clique sur un bouton avec l'id 'add' :
```javascript
$('#add').click(function(e) {
    e.preventDefault();
    // code    
});
```

* Récupération du nom dans l'input text :
```javascript
$('#name').val();
```

* Mise à jour du nom dans l'input text :
```javascript
$('#name').val('');
```

* Réagir au click sur une checkbox de tâche :
```javascript
$('body').on('click', '.label', function() {
    // code
});
```
* Faire un for each sur le modèle avec underscore js :
```javascript
_.each(todos, function(todo) {
    if (todo.done) {
        // code
    } else {
        // code
    }
});
```
* Ajout d'une tâche dans la liste de tâche :
```javascript
$('#tasks').append( '<li> ... </li>' );
```

* Pour récupérer une donnée dans un élément du dom, vous pouvez y ajouter un attribut data-* et le récupérer de la manière suivante :

```javascript
var id = $('#node').data('taskid');
```

et voici le template JS de départ (a mettre dans web/js/todo.js)

```javascript

var todos = [];
            
$(document).ready(function() {
    $('#add').click(function(e) {
        e.preventDefault();
        // code
    });
    $('body').on('click', '.label', function() {
        // code
    });
    $('#cleanup').click(function(e) {
        e.preventDefault();
        // code
    });   
});
```
