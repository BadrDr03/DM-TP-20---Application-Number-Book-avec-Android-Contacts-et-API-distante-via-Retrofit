# DM-TP-20---Application-Number-Book-avec-Android-Contacts-et-API-distante-via-Retrofit

Contexte général
Dans ce lab, il s’agit de développer une application Android nommée Number Book. Cette application a pour objectif de lire les contacts enregistrés dans le téléphone, de les afficher dans une interface mobile, puis de les envoyer vers un serveur distant afin de les stocker dans une base de données. Une fois les données enregistrées, l’application doit aussi permettre d’effectuer une recherche distante par nom ou par numéro de téléphone.
Ce lab est particulièrement intéressant car il met en relation plusieurs éléments importants du développement mobile moderne :
l’accès aux données système Android ;
la gestion des permissions ;
la communication client/serveur ;
la sérialisation JSON ;
l’utilisation de Retrofit pour consommer une API REST ;
la persistance des données dans une base distante.
L’objectif n’est pas seulement de produire une application fonctionnelle, mais aussi de comprendre le rôle de chaque bloc de code.

Objectifs pédagogiques
À la fin de ce lab, il devient possible de :
comprendre comment Android expose les contacts via ContentResolver ;
demander et gérer la permission READ_CONTACTS ;
récupérer les noms et numéros de téléphone stockés dans le mobile ;
afficher les données dans une liste moderne avec RecyclerView ;
envoyer des objets Java vers une API distante ;
récupérer des résultats JSON et les convertir automatiquement en objets Java ;
rechercher des contacts dans une base distante par nom ou par numéro ;
structurer un mini projet Android connecté à un backend.

Résultat attendu
À la fin de l’activité :
l’application demande l’accès aux contacts ;
les contacts du téléphone sont chargés et affichés ;
un bouton permet de synchroniser les contacts vers le serveur ;
la base distante contient les contacts envoyés ;
un champ de recherche permet d’interroger la base distante ;
les résultats trouvés sont affichés dans la liste.



Scénario de fonctionnement
Le fonctionnement global peut être résumé de la manière suivante :
l’utilisateur ouvre l’application ;
l’application demande l’autorisation d’accéder aux contacts ;
les contacts du téléphone sont lus à travers ContentResolver ;
les contacts sont affichés dans un RecyclerView ;
l’utilisateur clique sur Synchroniser ;
l’application envoie les contacts au backend via Retrofit ;
le serveur insère les données dans MySQL ;
l’utilisateur saisit un nom ou un numéro ;
l’application envoie une requête de recherche ;
le serveur retourne les résultats correspondants.Partie 1 — Conception de la base de données distante
Étape 1 — Créer la base de données
Créer la base numberbook.
CREATE DATABASE IF NOT EXISTS numberbook
CHARACTER SET utf8mb4
COLLATE utf8mb4_unicode_ci;

USE numberbook;
Explication
Cette commande crée un espace de stockage nommé numberbook.
Le choix de utf8mb4 permet de mieux gérer les caractères spéciaux, les noms accentués et les symboles.
Étape 2 — Créer la table contact
CREATE TABLE contact (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(150) NOT NULL,
    phone VARCHAR(50) NOT NULL,
    source VARCHAR(50) DEFAULT 'mobile',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
Explication détaillée
Cette table est conçue pour stocker les contacts envoyés depuis l’application Android.
id INT AUTO_INCREMENT PRIMARY KEY
Cette colonne représente l’identifiant unique du contact dans la base.
AUTO_INCREMENT signifie que MySQL génère automatiquement la valeur à chaque insertion.
name VARCHAR(150) NOT NULL
Cette colonne contient le nom du contact.
VARCHAR(150) indique qu’il s’agit d’une chaîne de caractères de taille variable, avec une longueur maximale de 150 caractères.
NOT NULL oblige à fournir une valeur.
phone VARCHAR(50) NOT NULL
Cette colonne stocke le numéro de téléphone.
Le type chaîne est préférable ici, car un numéro peut contenir +, espaces, parenthèses ou tirets.
source VARCHAR(50) DEFAULT 'mobile'
Cette colonne indique l’origine des données.
Dans ce lab, la valeur par défaut est mobile, ce qui signifie que le contact provient du téléphone.
created_at DATETIME DEFAULT CURRENT_TIMESTAMP
Cette colonne enregistre automatiquement la date et l’heure d’insertion.

Partie 2 — Développement du backend distant
Étape 3 — Organiser le projet serveur
Créer l’arborescence suivante :
numberbook-api/
│
├── config/
│   └── Database.php
│
├── model/
│   └── Contact.php
│
├── service/
│   └── ContactService.php
│
├── api/
│   ├── insertContact.php
│   ├── getAllContacts.php
│   └── searchContact.php
Explication
Cette organisation permet de séparer les responsabilités :
config contient la configuration technique ;
model contient les objets métier ;
service contient la logique d’accès aux données ;
api contient les points d’entrée HTTP.
Cette séparation rend le projet plus lisible et plus évolutif.
Étape 4 — Créer la connexion à la base
Fichier config/Database.php :
<?php
class Database {
    private $host = "localhost";
    private $dbName = "numberbook";
    private $username = "root";
    private $password = "";
    public $conn;

    public function getConnection() {
        $this->conn = null;

        try {
            $this->conn = new PDO(
                "mysql:host=" . $this->host . ";dbname=" . $this->dbName . ";charset=utf8mb4",
                $this->username,
                $this->password
            );
            $this->conn->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
        } catch (PDOException $exception) {
            echo "Erreur de connexion : " . $exception->getMessage();
        }

        return $this->conn;
    }
}
?>
Explication détaillée du code
class Database
Cette classe a pour rôle unique de gérer la connexion à la base de données.
private $host = "localhost";
Cela indique que le serveur MySQL se trouve sur la même machine que le serveur PHP.
private $dbName = "numberbook";
Nom de la base de données utilisée.
public function getConnection()
Cette méthode retourne l’objet PDO permettant d’exécuter les requêtes SQL.
new PDO(...)
Cette instruction crée la connexion à MySQL.
charset=utf8mb4
Cela force l’encodage en UTF-8 étendu.
PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION
Cette ligne indique à PDO de lever des exceptions en cas d’erreur SQL, ce qui facilite le débogage.
return $this->conn;
Cette ligne retourne la connexion créée pour qu’elle soit utilisée dans les services.
Étape 5 — Créer le modèle métier Contact
Fichier model/Contact.php :
<?php
class Contact {
    public $id;
    public $name;
    public $phone;
    public $source;
    public $created_at;
}
?>
Explication
Cette classe représente la structure logique d’un contact côté serveur.
Elle n’est pas strictement obligatoire dans une version minimale, mais elle améliore l’organisation du projet et clarifie les données manipulées.
Étape 6 — Créer le service ContactService
Fichier service/ContactService.php :
<?php
require_once __DIR__ . '/../config/Database.php';

class ContactService {
    private $conn;
    private $table = "contact";

    public function __construct() {
        $database = new Database();
        $this->conn = $database->getConnection();
    }

    public function insert($name, $phone, $source = "mobile") {
        $sql = "INSERT INTO " . $this->table . " (name, phone, source)
                VALUES (:name, :phone, :source)";
        $stmt = $this->conn->prepare($sql);
        return $stmt->execute([
            ':name' => $name,
            ':phone' => $phone,
            ':source' => $source
        ]);
    }

    public function getAll() {
        $sql = "SELECT * FROM " . $this->table . " ORDER BY name ASC";
        $stmt = $this->conn->prepare($sql);
        $stmt->execute();
        return $stmt->fetchAll(PDO::FETCH_ASSOC);
    }

    public function search($keyword) {
        $sql = "SELECT * FROM " . $this->table . "
                WHERE name LIKE :keyword OR phone LIKE :keyword
                ORDER BY name ASC";
        $stmt = $this->conn->prepare($sql);
        $stmt->execute([
            ':keyword' => '%' . $keyword . '%'
        ]);
        return $stmt->fetchAll(PDO::FETCH_ASSOC);
    }
}
?>
Explication détaillée
private $conn;
Contient la connexion PDO.
private $table = "contact";
Permet de centraliser le nom de la table utilisée.
__construct()
À l’instanciation du service, on crée la connexion à la base.
insert($name, $phone, $source = "mobile")
Cette méthode insère un contact.
Pourquoi utiliser une requête préparée ?
Parce qu’elle est plus propre et plus sûre qu’une concaténation directe.
$sql = "INSERT INTO contact (name, phone, source) VALUES (:name, :phone, :source)";
Ici, :name, :phone et :source sont des paramètres nommés.
$stmt->execute([...])
Cette instruction remplace les paramètres par les valeurs réelles, puis exécute la requête.
getAll()
Retourne tous les contacts triés par nom.
fetchAll(PDO::FETCH_ASSOC)
Retourne les résultats sous forme de tableau associatif, très pratique pour être transformé en JSON.
search($keyword)
Cette méthode recherche les contacts dont :
le nom contient le mot-clé ;
ou le numéro contient le mot-clé.
Le pattern utilisé est :
'%' . $keyword . '%'
Ce motif permet une recherche partielle.
Étape 7 — API d’insertion
Fichier api/insertContact.php :
<?php
header("Content-Type: application/json");
require_once __DIR__ . '/../service/ContactService.php';

if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    $data = json_decode(file_get_contents("php://input"), true);

    if (!isset($data['name']) || !isset($data['phone'])) {
        echo json_encode([
            "success" => false,
            "message" => "Champs manquants"
        ]);
        exit;
    }

    $service = new ContactService();
    $ok = $service->insert($data['name'], $data['phone'], "mobile");

    echo json_encode([
        "success" => $ok,
        "message" => $ok ? "Contact inséré avec succès" : "Erreur d'insertion"
    ]);
}
?>
Explication détaillée du code
header("Content-Type: application/json");
Indique que la réponse sera envoyée au format JSON.
$_SERVER['REQUEST_METHOD'] === 'POST'
Vérifie que l’appel reçu est bien une requête POST.
file_get_contents("php://input")
Lit le corps brut de la requête HTTP.
json_decode(..., true)
Transforme le JSON reçu en tableau PHP associatif.
Vérification des champs
Avant l’insertion, le code vérifie que name et phone sont présents.
json_encode([...])
Transforme la réponse PHP en JSON pour qu’Android puisse la lire facilement.
Étape 8 — API de lecture complète
Fichier api/getAllContacts.php :
<?php
header("Content-Type: application/json");
require_once __DIR__ . '/../service/ContactService.php';

$service = new ContactService();
$result = $service->getAll();

echo json_encode($result);
?>
Explication
Ce script est simple :
il crée le service ;
il récupère tous les contacts ;
il les renvoie au format JSON.
Étape 9 — API de recherche
Fichier api/searchContact.php :
<?php
header("Content-Type: application/json");
require_once __DIR__ . '/../service/ContactService.php';

if (!isset($_GET['keyword'])) {
    echo json_encode([]);
    exit;
}

$keyword = $_GET['keyword'];

$service = new ContactService();
$result = $service->search($keyword);

echo json_encode($result);
?>
Explication détaillée
$_GET['keyword']
Récupère le mot-clé envoyé dans l’URL.
Exemple :
searchContact.php?keyword=ali
Pourquoi retourner [] si rien n’est fourni ?
Parce qu’une réponse JSON vide est plus propre qu’une erreur brute dans ce contexte pédagogiqueÉtape 10 — Créer le projet
Créer un projet Android Studio nommé NumberBook.
Choisir une activité vide.
Étape 11 — Ajouter les permissions
Dans AndroidManifest.xml :
<uses-permission android:name="android.permission.READ_CONTACTS" />
<uses-permission android:name="android.permission.INTERNET" />
Explication
READ_CONTACTS
Autorise l’application à lire les contacts du téléphone.
INTERNET
Autorise l’application à envoyer et recevoir des données via le réseau.
Sans ces permissions, l’application ne pourra ni lire les contacts ni communiquer avec le backend.
Étape 12 — Ajouter les dépendances Gradle
implementation 'com.squareup.retrofit2:retrofit:2.11.0'
implementation 'com.squareup.retrofit2:converter-gson:2.11.0'
implementation 'androidx.recyclerview:recyclerview:1.3.2'
implementation 'com.google.android.material:material:1.11.0'
Explication
Retrofit
Bibliothèque qui simplifie les appels HTTP.
converter-gson
Permet de convertir automatiquement :
un objet Java en JSON ;
un JSON reçu en objet Java.
RecyclerView
Permet d’afficher efficacement une liste de contacts.Étape 13 — Créer l’écran principal
Fichier activity_main.xml :
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="16dp">

    <Button
        android:id="@+id/btnLoadContacts"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Charger les contacts" />

    <Button
        android:id="@+id/btnSyncContacts"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Synchroniser vers le serveur"
        android:layout_marginTop="8dp" />

    <EditText
        android:id="@+id/etKeyword"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:hint="Rechercher par nom ou numéro"
        android:layout_marginTop="12dp" />

    <Button
        android:id="@+id/btnSearch"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Rechercher"
        android:layout_marginTop="8dp" />

    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/recyclerViewContacts"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1"
        android:layout_marginTop="12dp" />
</LinearLayout>
Explication détaillée
Bouton btnLoadContacts
Lance la lecture des contacts du téléphone.
Bouton btnSyncContacts
Déclenche la synchronisation avec le serveur.
EditText etKeyword
Permet à l’utilisateur de saisir un nom ou un numéro.
Bouton btnSearch
Lance la recherche distante.
RecyclerView
Affiche les contacts.
Le choix de layout_height="0dp" avec layout_weight="1" permet à la liste d’occuper l’espace restant.Étape 14 — Classe Contact.java
package com.example.numberbook;

public class Contact {
    private int id;
    private String name;
    private String phone;
    private String source;
    private String created_at;

    public Contact() {
    }

    public Contact(String name, String phone) {
        this.name = name;
        this.phone = phone;
    }

    public int getId() {
        return id;
    }

    public String getName() {
        return name;
    }

    public String getPhone() {
        return phone;
    }

    public String getSource() {
        return source;
    }

    public String getCreated_at() {
        return created_at;
    }

    public void setId(int id) {
        this.id = id;
    }

    public void setName(String name) {
        this.name = name;
    }

    public void setPhone(String phone) {
        this.phone = phone;
    }

    public void setSource(String source) {
        this.source = source;
    }

    public void setCreated_at(String created_at) {
        this.created_at = created_at;
    }
}
Explication
Cette classe représente un contact côté Android.
Pourquoi créer ce modèle ?
Parce que Retrofit et Gson vont convertir automatiquement les réponses JSON du backend en objets Contact.
Constructeur vide
Nécessaire pour certaines opérations de désérialisation.
Constructeur Contact(String name, String phone)
Utile pour créer rapidement un contact à partir des données lues dans le téléphone.
Étape 15 — Classe ApiResponse.java
package com.example.numberbook;

public class ApiResponse {
    private boolean success;
    private String message;

    public boolean isSuccess() {
        return success;
    }

    public String getMessage() {
        return message;
    }
}
Explication
Cette classe représente la structure JSON de réponse d’une opération comme l’insertion.
Exemple de réponse serveur :
{
  "success": true,
  "message": "Contact inséré avec succès"
}

Étape 16 — Interface ContactApi.java
package com.example.numberbook;

import java.util.List;

import retrofit2.Call;
import retrofit2.http.Body;
import retrofit2.http.GET;
import retrofit2.http.POST;
import retrofit2.http.Query;

public interface ContactApi {

    @POST("insertContact.php")
    Call<ApiResponse> insertContact(@Body Contact contact);

    @GET("getAllContacts.php")
    Call<List<Contact>> getAllContacts();

    @GET("searchContact.php")
    Call<List<Contact>> searchContacts(@Query("keyword") String keyword);
}
Explication détaillée
Cette interface décrit l’API REST côté Android.
@POST("insertContact.php")
Indique que la méthode appelle le point d’entrée insertContact.php.
@Body Contact contact
L’objet Java Contact est automatiquement converti en JSON.
Call<ApiResponse>
Indique que le serveur renverra un objet ApiResponse.
@GET("getAllContacts.php")
Récupère tous les contacts.
Call<List<Contact>>
Le backend renvoie une liste JSON, donc Retrofit la convertit en List<Contact>.
@Query("keyword")
Ajoute un paramètre dans l’URL.
Exemple généré :
searchContact.php?keyword=ali
Étape 17 — Créer RetrofitClient.java
package com.example.numberbook;

import retrofit2.Retrofit;
import retrofit2.converter.gson.GsonConverterFactory;

public class RetrofitClient {

    private static final String BASE_URL = "http://192.168.1.10/numberbook-api/api/";
    private static Retrofit retrofit;

    public static Retrofit getClient() {
        if (retrofit == null) {
            retrofit = new Retrofit.Builder()
                    .baseUrl(BASE_URL)
                    .addConverterFactory(GsonConverterFactory.create())
                    .build();
        }
        return retrofit;
    }
}
Explication détaillée
BASE_URL
C’est l’adresse racine de l’API.
Pourquoi terminer par / ?
Parce que Retrofit construit ensuite les URLs relatives comme insertContact.php.
GsonConverterFactory.create()
Permet la conversion automatique JSON ↔ objets Java.
if (retrofit == null)
Met en place un singleton simple pour éviter de recréer plusieurs insta
