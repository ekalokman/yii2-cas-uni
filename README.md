yii2-cas-uni
=============

Yii2 library for authentication by CAS,
using the library [phpCAS](https://github.com/apereo/phpCAS).

Usage
-----

1. Add this to the project with `composer require ekalokman/yii2-auth-uni`

2. Configure the Yii2 application, e.g. in `backend/config/main.php` :

    ```
    return [
        ...
        'modules' => [
            'cas' => [
                'class' => 'silecs\yii2auth\cas\CasModule',
                'config' => [
                    'host' => 'ssoserver.example.com', //insert your own host
                    'port' => '443', //insert your own port
                    'path' => '/cas',
                    'returnUrl' => '', //insert your own return url
                    // optional parameters
                    'certfile' => '', // empty, or path to a SSL cert, or false to ignore certs
                    'debug' => true, // will add many logs into X/runtime/logs/cas.log
                ],
            ],
    ```

3. Add actions that use this CAS module, in `SiteController` :

    ```
    public function actionLogin()
    {
        if (!Yii::$app->user->isGuest) {
            return $this->goHome();
        }
        return $this->redirect(['/cas/auth/login']);
    }

    public function actionLogout()
    {
        if (Yii::$app->user->isGuest) {
            return $this->redirect(['/cas/auth/logout']);
        }
        return $this->goHome();
    }
    ```

3. Add actions that use casAuthenticate to check user is student or staff and register new user, in `common/models/LoginCas.php` :

    ```
   <?php
    namespace common\models;

    use Yii;
    use yii\base\Model;
    use phpCAS;
    use yii\helpers\Url;
    use common\models\User;
    use common\models\Student;
    use backend\models\StudentSt;

    /**
    * Login form
    */
    class LoginCas extends Model
    {
        private $_user;

        /**
        * @inheritdoc
        */
        public function rules()
        {
            return [
            ];
        }

        /**
        * Logs in a user using the provided username and password.
        *
        * @return boolean whether the user is logged in successfully
        */
        public function login()
        {
            $user = $this->casAuthenticate();
        
            if ($user) {
                $user = $this->getUser($user);
                return Yii::$app->user->login($user, false ? 3600 * 24 * 30 : 0);
            } else {
                return false;
            }
        }

        /**
        * Finds user by [[username]]
        *
        * @return User|null
        */
        protected function getUser($user)
        {
            if ($this->_user === null) {
                $this->_user = UserCas::findByUsername($user);
            }

            return $this->_user;
        }

        public static function casAuthenticate($username)
        {

            //may change checking below if want to check staff only
            $cStudentSt=new StudentSt(); //for checking username whether student or staff
            $StuData=$cStudentSt->getDataSt($username);

            $baseUrl = Url::base(true);
            $baseUrl = Url::base();

            //for student screen
            if ($baseUrl == '/stu'){
            
                if($StuData){ // for student checking username is student: true and not null

                    $userCas = Student::findByUsername($username);

                    if (empty($userCas)) {

                            $con = \Yii::$app->db;
                            $attributes = [
                                'username' => $username,
                                'auth_key' => Yii::$app->security->generateRandomString(),
                                'status' => '10',
                                'created_at' => time(),
                                'updated_at' => time()
                            ];
                            $con->createCommand()->insert('quest.qst_student', $attributes)->execute();
                        // $user = $this->getUser();

                    }

                }else{ //student is false

                    Yii::$app->user->logout();
                    unset($_COOKIE);

                    Yii::$app->user->logout();

                    header("cache-Control: no-store, no-cache, must-revalidate");
                    header("cache-Control: post-check=0, pre-check=0", false);
                    header("Pragma: no-cache");
                    header("Expires: Sat, 26 Jul 1997 05:00:00 GMT");
                
                    echo "<script>alert('Unauthorized access! This application only allow for IIUM student. \\nKindly contact the Administrator if any issue.');
                                window.location = 'https://cas.iium.edu.my:8448/cas/logout';
                    </script>";
                    exit;

                }

            //for staff screen
            }else{ 

                if(empty($StuData)){ //for staff checking username is student: false and null

                    $userCas = User::findByUsername($username); //checking staff is a

                    if ($userCas === null || $userCas->username === null) {

                        $con = \Yii::$app->db;
                        $attributes = [
                            'username' => $username,
                            'auth_key' => Yii::$app->security->generateRandomString(),
                            'status' => '10',
                            'created_at' => time(),
                            'updated_at' => time()
                        ];
                        $con->createCommand()->insert('table.user', $attributes)->execute(); //insert to your own table

                    }

                }else{//student is true and not staff

                    Yii::$app->user->logout();

                    unset($_COOKIE);

                    Yii::$app->user->logout();

                    header("cache-Control: no-store, no-cache, must-revalidate");
                    header("cache-Control: post-check=0, pre-check=0", false);
                    header("Pragma: no-cache");
                    header("Expires: Sat, 26 Jul 1997 05:00:00 GMT");
                
                    echo "<script>alert('Unauthorized access! This application only allow for IIUM staff. \\nKindly contact the Administrator if any issue.');
                                window.location = 'https://cas.iium.edu.my:8448/cas/logout';
                    </script>";
                    exit;

                }

            }
        }

    }

    ```


Notes
-----

The `user` component that implements `yii\web\IdentityInterface`
will be used to fetch the local profile after querying the CAS server.
It means that if `User` is the App component and CAS returns a username of "bibendum",
the authentication will be successful if and only if
the result of `User::findIdentity("bibendum")` is not null.

The action path '/cas/auth/login' starts with the alias of the module,
as defined in the application configuration, e.g.
`'cas'` in `'modules' => [ 'cas' => [ ... ] ]`.


### Testing with a CAS container

Here are some instructions on deploying a Docker CAS server
to test this library.
This procedure will use the CAS interface of a Shibboleth instance.
This was tested on Debian Stretch and Buster (testing).

1. Install `docker` from the extra repository at docker.io
   (I had errors with the older docker from the official Debian repository).

2. Install `docker-compose` either from Debian or docker.io.

3. Git clone https://hub.docker.com/r/unicon/shibboleth-idp/
   If using an old docker-compose, then chekout 3c29f10
   because later commits require a too recent feature.

4. Modify `docker-compose.yml` so that the container won't try to use the port 80,
   so replace `"80:80"` with `"8080:80"`.

5. If your local Yii2 application is not using HTTPS,
   modify `idp/shibboleth-idp/conf/cas-protocol.xml`
   to replace `c:regex="https://idptestbed/.*"` by `c:regex="https?://idptestbed/.*"`.

6. Add `127.0.0.1 idptestbed` to `/etc/hosts`, as root.

7. Configure your Yii2 application to use:

        'host' => 'idptestbed',
        'port' => '443',
        'path' => '/idp/profile/cas',
        'certfile' => false,
        'debug' => true,

8. Start the containers:

        docker-compose build
        docker-compose run

9. Go to the login page of your Yii2 app.

10. Ctrl-C in the containers termainal to end them.

You can modify `ldap/users.ldif` if you want to add users to the CAS.
Don't forget to rebuild the Docker images after this.
