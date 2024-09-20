# 🚀 Intégration de Google OAuth avec Symfony

## Introduction
Ce guide vous montre comment intégrer facilement l'authentification via Google OAuth2 dans votre projet **Symfony 6.4** 🛠️.
Cela permet à vos utilisateurs de se connecter avec leurs comptes Google de manière sécurisée 🔒.
Pour plus de détails sur la mise en place de votre environnement Symfony avec Docker, consultez [Mon Repository GitHub](https://github.com/abdelhakmireda/Environnement-D-veloppement-Docker-Symfony) 📦.

## 📋Prérequis
Avant de commencer, assurez-vous d'avoir les éléments suivants :

- **Composer** : pour la gestion des dépendances.
- **Symfony CLI** : pour la création et la gestion du projet Symfony.
- **MySQL** : pour la base de données.
- **PHP >= 8.0.2** : pour faire fonctionner Symfony 🐘.

## 📦Installation des Dépendances

Installez les dépendances nécessaires en utilisant les commandes suivantes :

```bash
composer require knpuniversity/oauth2-client-bundle
composer require league/oauth2-google
```
## 🛠️  Configuration du Compte Google

1. **Créez un projet** dans votre compte Google Cloud.
2. **Accédez à Google API Console** pour obtenir vos identifiants OAuth.
3. **Ajoutez les variables d’environnement** suivantes dans votre fichier `.env` :

```dotenv
OAUTH_GOOGLE_CLIENT_ID=Votre_Client_ID
OAUTH_GOOGLE_CLIENT_SECRET=Votre_Client_Secret
OAUTH_GOOGLE_REDIRECT_URI=https://127.0.0.1:8000/connect/google/check
```
## ⚙️ Configuration de `knpu_oauth2_client.yaml`

Ajoutez la configuration suivante dans le fichier `config/packages/knpu_oauth2_client.yaml` :

```yaml
knpu_oauth2_client:
    clients:
        google:
            type: google
            client_id: '%env(OAUTH_GOOGLE_CLIENT_ID)%'
            client_secret: '%env(OAUTH_GOOGLE_CLIENT_SECRET)%'
            redirect_route: connect_google_check
            redirect_params: {}
```
## 🧑‍💻 Création du Contrôleur Google

1. **Générez le contrôleur** avec la commande suivante🖥️ :

```bash
php bin/console make:controller Google
```
## Ensuite, Remplacez le contenu du contrôleur par le code suivant :

```php
<?php

namespace App\Controller;

use KnpU\OAuth2ClientBundle\Client\ClientRegistry;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\Routing\Annotation\Route;
use League\OAuth2\Client\Provider\Exception\IdentityProviderException;
use App\Entity\User;
use Symfony\Component\PasswordHasher\Hasher\UserPasswordHasherInterface;
use Doctrine\ORM\EntityManagerInterface;
use Symfony\Component\Security\Http\Authentication\UserAuthenticatorInterface;
use App\Security\LoginFormAuthenticator;

class GoogleController extends AbstractController
{
    #[Route('/connect/google', name: 'connect_google_start')]
    public function connectAction(ClientRegistry $clientRegistry)
    {
        // Vérifiez si l'utilisateur est déjà connecté
        if ($this->getUser()) {
            return $this->redirectToRoute('dashboard');
        }

        // Redirigez l'utilisateur vers Google pour l'authentification
        return $clientRegistry->getClient('google')->redirect(['profile', 'email'], []);
    }

    #[Route('/connect/google/check', name: 'connect_google_check')]
    public function connectCheckAction(
        Request $request,
        ClientRegistry $clientRegistry,
        UserPasswordHasherInterface $userPasswordHasher,
        EntityManagerInterface $entityManager,
        UserAuthenticatorInterface $userAuthenticator,
        LoginFormAuthenticator $authenticator
    ) {
        // Vérifiez si l'utilisateur est déjà connecté
        if ($this->getUser()) {
            return $this->redirectToRoute('app_home');
        }

        // Récupérer le client Google
        $client = $clientRegistry->getClient('google');

        try {
            // Récupérer l'utilisateur Google
            $googleUser = $client->fetchUser();
            $userData = $googleUser->toArray();
            $email = $userData['email'] ?? null;

            if (!$email) {
                throw new \Exception('Email not provided by Google.');
            }

            // Vérifiez si l'utilisateur existe déjà dans la base de données
            $existingUser = $entityManager->getRepository(User::class)
                ->findOneBy(['email' => $email]);

            if ($existingUser) {
                // Authentifiez l'utilisateur existant
                return $userAuthenticator->authenticateUser(
                    $existingUser,
                    $authenticator,
                    $request
                );
            }

            // Créez un nouvel utilisateur si aucun n'existe
            $user = new User();
            $user->setEmail($email);
            $user->setPassword(
                $userPasswordHasher->hashPassword($user, $googleUser->getId())
            );

            // Enregistrez le nouvel utilisateur
            $entityManager->persist($user);
            $entityManager->flush();

            // Authentifiez le nouvel utilisateur
            return $userAuthenticator->authenticateUser(
                $user,
                $authenticator,
                $request
            );
        } catch (IdentityProviderException $e) {
            // Gérer les erreurs d'authentification
            $this->addFlash('error', 'Erreur lors de l\'authentification : ' . $e->getMessage());
            return $this->redirectToRoute('app_login');
        } catch (\Exception $e) {
            // Gérer les autres erreurs
            $this->addFlash('error', 'Erreur : ' . $e->getMessage());
            return $this->redirectToRoute('app_login');
        }
    }
}
```
# 🎉  Conclusion
Vous avez maintenant configuré Google OAuth2 dans Symfony 6.4 ! ✅ Vos utilisateurs peuvent s'authentifier en toute simplicité avec leurs comptes Google, offrant une meilleure expérience utilisateur 🎉. N'hésitez pas à ajouter des personnalisations pour enrichir davantage votre application Symfony 💡.

Bonne intégration ! 😊
