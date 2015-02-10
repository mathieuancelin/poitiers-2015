TP Java EE : la TODO list
=================================

Nous allons réaliser aujourd'hui une simple `TODO list` en utilisant diverses technologies Java EE : EJB, JAX-RS (client + server), JPA, CDI ainsi qu'une bonne dose de Javascript. L'application à réaliser ressemblera à [quelque chose comme ça](http://ancelin.org/example/). Cette application exposera des services REST qui seront consommés par une IHM en Javascript.

Commencez par télécharger le squelette du TP [ici](https://raw.githubusercontent.com/mathieuancelin/poitiers-2015/master/todo-starter.zip). Si vous utilisez un poste de la fac, il faudra installer la dernière version de Netbeans + Glassfish.

Le projet contient déjà le minimum vital pour avoir une application qui fonctionne. De plus le projet contient une IHM de démo pour tester vos service ainsi qu'un outil de diagnostique pour vérifier que votre implémentation tient la route.

I. Partie Serveur
--------------------------

Tout d'abord il faut commencer par mettre en place tout la tuyauterie nécessaire pour l'application.

Il faut commencer par activer JAX-RS dans l'application. Pour celà créez un classe com.foo.bar.todo.App.java comme suivant :

```java
package com.foo.bar.todo;

import javax.ws.rs.ApplicationPath;
import javax.ws.rs.core.Application;

@ApplicationPath("api") // toutes les api rest seront dispo à /Todo/api/*
public class App extends Application {
}
```

Il va falloir ensuite être capable de communiquer avec la base de données. Pour celà nous avons besoin de récupérer une instance de l'`EntityManager` courant. Et comme nous voulons nous en service en utilisant `@Inject EntityManager em;` il va falloir aider CDI. Pour celà créer une classe com.foo.bar.todo.EmProducer.java comme suivant :

```java
package com.foo.bar.todo;

import javax.enterprise.context.ApplicationScoped;
import javax.enterprise.inject.Produces;
import javax.persistence.EntityManager;
import javax.persistence.PersistenceContext;

@ApplicationScoped
public class EmProducer {

    @PersistenceContext @Produces // on inject avec @PersistenceContext et on le donne à CDI avec @Produces
    private EntityManager em;
}
```

Maintenant on peut s'attaquer au code métier de notre application

Premièrement, nous allons créer la couche de données de l'application. Pour celà créer une classe com.foo.bar.todo.models.Task.java

```java
package com.foo.bar.todo.models;

import java.io.Serializable;
import java.util.List;
import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;
import javax.xml.bind.annotation.XmlRootElement;
import javax.enterprise.inject.spi.CDI;
import javax.persistence.EntityManager;

@Entity @XmlRootElement
public class Task implements Serializable {
    
    @Id @GeneratedValue(strategy= GenerationType.AUTO)
    private Long id;
    private Boolean done;
    private String name;
  
    getters, setters, etc ...

```

Afin de coller à une conception orientée objet, nous allons éviter de créer des couches de services ou de DAO permettant l'accès au données et nous allons coder toutes les méthodes concernant la classe Task dans la classe Task.
Une des particularité des beans JPA est qu'ils ne peuvent pas être injectés. Du coup il n'est pas possible de récupérer l'entity manager courant depuis un bean JPA. Nous allons donc utiliser une astuce pour le récupérer. 

Dans votre classe Task, ajouter la méthode suivante 

```java
private static EntityManager em() {
    // ici on utiliser CDI.current() permettant de récupérer le context CDI depuis un context non managé par CDI
    // nous sélectionnons ensuite le bean que nous souhaitons récupérer et récupérons l'instance courante.
    return CDI.current().select(EntityManager.class).get();
}
```

Maintenant il nous reste à coder les différentes méthodes permettant d'intéragir avec la base de données

```java
public Task save() {
    if (em().contains(this)) {
        return em().merge(this);
    } else {
        em().persist(this);
        return em().find(getClass(), id);
    }
}

public static int deleteAll() {
    return em().createQuery("delete from Task").executeUpdate();
}

public static int deleteAllDone() {
    return em().createQuery("delete from Task u where u.done = true").executeUpdate();
}

public static Task findById(long id) {
    return em().find(Task.class, id);
}

public void delete() {
    em().remove(findById(id));
}

public static List<Task> findAll() {
    return em().createQuery("select u from Task u", Task.class).getResultList();
}

public static List<Task> findAllDone() {
    return em().createQuery("select u from Task u where u.done = true", Task.class).getResultList();
}

public static List<Task> findAllToDo() {
    return em().createQuery("select u from Task u where u.done = false", Task.class).getResultList();
}

public static Long count() {
    Long l = Long.parseLong(em().createQuery("select count(u) from Task u").getSingleResult().toString());
    if (l == null || l < 0) {
        return 0L;
    }
    return l;
}
```

Maintenant, il faut créer les services REST correspondant. Vous pouvez créer une classe com.foo.bar.todo.services.TodoServices.java

Cette classe doit répondre au contrat suivant :

* GET    /Todo/api/tasks       => récupération toutes les instances de tâches
* POST   /Todo/api/tasks       => création une nouvelle tâche 
* PUT    /Todo/api/tasks/:id   => mise à jour d'une tâche
* DELETE /Todo/api/tasks       => supression de toutes les tâches
* DELETE /Todo/api/tasks/done  => supression des tâches finies
* GET    /Todo/api/tasks/:id   => récupération d'une tâche
* DELETE /Todo/api/tasks/:id   => suppression d'une tâche en particulier

Voici un exemple de classe répondant à ce contrat

```java
package com.foo.bar.todo.services;

import com.foo.bar.todo.models.Task;
import java.util.List;
import javax.ejb.Stateless;
import javax.ws.rs.Consumes;
import javax.ws.rs.DELETE;
import javax.ws.rs.FormParam;
import javax.ws.rs.GET;
import javax.ws.rs.NotFoundException;
import javax.ws.rs.POST;
import javax.ws.rs.PUT;
import javax.ws.rs.Path;
import javax.ws.rs.PathParam;
import javax.ws.rs.Produces;
import javax.ws.rs.core.MediaType;

@Stateless
@Path("tasks")
public class TodoServices {

    @GET
    @Produces(MediaType.APPLICATION_JSON)
    public List<Task> allTasks() {
        // TODO : à implémenter
    }
    
    @Path("done") @DELETE
    @Produces(MediaType.APPLICATION_JSON)
    public void deleteAllDone() {
        // TODO : à implémenter
    }
    
    @Path("{id}") @PUT
    @Produces(MediaType.APPLICATION_JSON)
    @Consumes(MediaType.APPLICATION_FORM_URLENCODED)
    public Task changeTaskState(@PathParam("id") Long id, @FormParam("done") Boolean done) {
        // TODO : à implémenter
    }
    
    @POST
    @Produces(MediaType.APPLICATION_JSON)
    @Consumes(MediaType.APPLICATION_FORM_URLENCODED)
    public Task createTask(@FormParam("name") String name) {
        // TODO : à implémenter
    }
    
    @Path("{id}") @GET
    @Produces(MediaType.APPLICATION_JSON)
    public Task getTask(@PathParam("id") Long id) {
        // TODO : à implémenter
    }
    
    @Path("{id}") @DELETE
    @Produces(MediaType.APPLICATION_JSON)
    public void deleteTask(@PathParam("id") Long id) {
        // TODO : à implémenter
    }
    
    @DELETE
    @Produces(MediaType.APPLICATION_JSON)
    public void deleteAll() {
        // TODO : à implémenter
    }
}

```

Nous pouvons maintenant tester nos services. Pour celà, utilisez `curl` ou un plugin navigateur type `POSTman`


```sh
$ curl -X GET http://localhost:8080/Todo/api/tasks
[]
```

```sh
$ curl -X POST -d 'name=Hello' http://localhost:8080/Todo/api/tasks
{"name":"Hello", "id":1, "done":false}
$ curl -X POST -d 'name=Hello2' http://localhost:8080/Todo/api/tasks
{"name":"Hello2", "id":2, "done":false}
```

```sh
$ curl -X GET http://localhost:8080/Todo/api/tasks
[{"name":"Hello", "id":1, "done":false}, {"name":"Hello2", "id":2, "done":false}]
```

```sh
$ curl -X PUT -d 'done=true' http://localhost:8080/Todo/api/tasks/1
{"name":"Hello", "id":1, "done":true}
```

```sh
$ curl -X GET http://localhost:8080/Todo/api/tasks
[{"name":"Hello", "id":1, "done":true}, {"name":"Hello2", "id":2, "done":false}]
```

```sh
$ curl -X DELETE http://localhost:8080/Todo/api/tasks/done
```

```sh
$ curl -X GET http://localhost:8080/Todo/api/tasks
[{"name":"Hello2", "id":2, "done":false}]
```

```sh
$ curl -X DELETE http://localhost:8080/Todo/api/tasks
```

```sh
$ curl -X GET http://localhost:8080/Todo/api/tasks
[]
```

Nous pouvons maintenant passer à la partie client

II. Partie Client
---------------------------

Nous allons coder la partie client en Javascript en utiliser JQuery. Vous pouvez utiliser n'importe quel autre framework JS du moment que vous savez l'utiliser. Je vous conseille cependant de regarder du côté d'Angular et de React.

Vous devrez écrire votre code javascript dans `web/js/todo.js`. Commencez par effacer les lignes 

```javascript
var el = ...;
$('#todo').html(el);
```

qui affichent le message `Votre application ici ;-)` et cachent un template de base pour votre application. La partie HTML de l'application est disponible dans `web/index.html`.

Voici le template html de l'application :

```html
<div class="col-md-4">
  <h3>Todo List</h3>
  <div>
    <div class="row">
      <form role="form">
        <div class="form-group col-md-10">
          <input id="name" placeholder="What do you have to do ?" type="text" class="form-control">
        </div>
        <div class="form-group">
          <div class="btn-group">
            <button id="add" type="button" class="btn btn-success">
              <span class="glyphicon glyphicon-floppy-saved"></span>
            </button>
            <button id="cleanup" type="button" class="btn btn-danger">
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

* Lors du clic sur le bouton 'Add', si le texte entré dans l'input text du formulaire n'est pas vide, créer une tâche (via le service REST approprié) et l'ajouter dans la liste de tâche. La partie HTML est disponible dans le fichier `web/index.html`.

```html
<li className="list-group-item">
    <div className="row">
        <div className="col-md-10">
            <span>Nom de la tache</span>
        </div>
        <div className="col-md-2">
            <span class="label label-success">Done</span>
        </div>
    </div>
</li>
```

* Lors du clic sur une tâche, marquer la tâche comme effectuée (via le service REST approprié) et appliqué le style `label-success` sur le `span` de la tâche afin qu'elle apparaisse barrée.
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


III. Micro services
-------------------------

Pour cette partie, vous développerez la même application, à la différence que celle-ci sera séparée en trois morceaux. 

Commencez par télécharger les squelettes de ces trois applications [ici](https://raw.githubusercontent.com/mathieuancelin/jpoitiers-2015/master/backuser-starter.zip), [ici](https://raw.githubusercontent.com/mathieuancelin/jpoitiers-2015/master/backtask-starter.zip) et [ici](https://raw.githubusercontent.com/mathieuancelin/jpoitiers-2015/master/front-starter.zip).

Ici, l'application Frontend (qui contient l'IHM) se chargera de consommer divers services depuis deux applications backend afin de créer une `TODO list` multi utilisateurs. Comme lors de la partie précedente, l'application frontend contient une démo ainsi qu'un outil de diagnostique.

Voici les services à implémenter côté frontend. Vous êtes libre d'implémenter les services backend comme vous voudrez. Cependant votre application frontend doit absolument communiquer avec les applications frontend grâce à des services REST (vous utiliserez le client HTTP fourni avec JAX-RS). La différence majeure de cette version est qu'il va devoir être nécessaire de gérer les utilisateurs qui créé des tâches. Pour celà il faudra créer un service de gestion d'utilisateurs et il faudra rajouter un id utilisateur dans chaque tâche. Il faudra également ajouter un sélecteur d'utilisateur dans l'IHM et afficher le nom de l'utilisateur qui à créer la tâche pour chaque tâche.

* GET    /TodoFrontend/api/users       => récupération tous les utilisateurs du système
* GET    /TodoFrontend/api/tasks       => récupération toutes les instances de tâches
* POST   /TodoFrontend/api/tasks + param userId et name dans le body     => création une nouvelle tâche 
* PUT    /TodoFrontend/api/tasks/:id + param userId et param done dans le body => mise à jour d'une tâche
* DELETE /TodoFrontend/api/tasks       => supression d'une tâche en particulier
* DELETE /TodoFrontend/api/tasks/done  => supression des tâches finies
* GET    /TodoFrontend/api/tasks/:id?userId=:userId   => récupération d'une tâche
* DELETE /TodoFrontend/api/tasks/:id?userId=:userId   => suppression d'une tâche

Voici un exemple de service dans le Frontend 

```java
@GET @Path("tasks")
@Produces(MediaType.APPLICATION_JSON)
public JsonArray getAllTasks() {
    JsonArrayBuilder builder = Json.createArrayBuilder();
    for (User user : users()) {
        JsonArray arr = client.target("http://localhost:8080/BackendTasks/api/").path(user.getEmail()).path("/tasks").request().get(JsonArray.class);
        for (JsonValue value : arr) {
            builder = builder.add(value);
        }
    }
    return builder.build();
}
```

Je vous conseille de lire un peu de documentation sur le client HTTP de JAX-RS

https://docs.oracle.com/javaee/7/tutorial/doc/jaxrs-client001.htm
https://docs.oracle.com/javaee/7/api/index.html?javax/ws/rs/client/package-summary.html
