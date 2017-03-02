Пример Менеджер Закладок Часть 2
################################

После завершения :doc:`первой части данного руководства
</tutorials-and-examples/bookmarks/intro>` у вас должно быть очень простое
приложение для закладок. В этой главе мы добавим аутентификацию и ограничения
доступа к закладкам, чтобы каждый пользователь мог видеть/изменять только те
закладки, которые создавал непосредственно он.

Вход пользователя
=================

В CakePHP аутентификация обрабатывается :doc:`/controllers/components`.
Компоненты можно представить как повторно используемые куски кода Контроллера
для реализации какой-либо функциональности. Компоненты также могут цепляться
к жизненному циклу событий Контроллера и взаимодействовать с вашим 
приложением таким способом. Для начала мы добавим :doc:`Компонент Auth
</controllers/components/authentication>` в наше приложение. Мы в достаточной
степени хотим, чтобы каждый метод требовал аутентификацию, так что, мы 
добавим AuthComponent в наш AppController::

    // В src/Controller/AppController.php
    namespace App\Controller;

    use Cake\Controller\Controller;

    class AppController extends Controller
    {
        public function initialize()
        {
            $this->loadComponent('Flash');
            $this->loadComponent('Auth', [
                'authenticate' => [
                    'Form' => [
                        'fields' => [
                            'username' => 'email',
                            'password' => 'password'
                        ]
                    ]
                ],
                'loginAction' => [
                    'controller' => 'Users',
                    'action' => 'login'
                ],
                'unauthorizedRedirect' => $this->referer() // Если не авторизованы, то возвращаем на страницу, где только что были
            ]);

            // Разрешение экшена display, чтобы наш контроллер pages
            // продолжал работать.
            $this->Auth->allow(['display']);
        }
    }

Мы только что сообщили CakePHP, что мы хотим загрузить компоненты ``Flash`` и
``Auth``. В дополнение мы кастомизировали конфигурацию компонента Auth, таким
образом, что наша таблица users использует ``email`` в качестве имени
пользователя. Теперь, если вы попробуете перейти по любому URL, то будете
переброшены наURL **/users/login**, который покажет ошибку, так как мы еще не
написали необходимый код. Давайте создадим экшен login::

    // В src/Controller/UsersController.php
    public function login()
    {
        if ($this->request->is('post')) {
            $user = $this->Auth->identify();
            if ($user) {
                $this->Auth->setUser($user);
                return $this->redirect($this->Auth->redirectUrl());
            }
            $this->Flash->error('Ваше имя пользователя или пароль не верны.');
        }
    }
    
И в **src/Template/Users/login.ctp** добавьте следующее::

    <h1>Вход</h1>
    <?= $this->Form->create() ?>
    <?= $this->Form->input('email') ?>
    <?= $this->Form->input('password') ?>
    <?= $this->Form->button('Войти') ?>
    <?= $this->Form->end() ?>

Теперь, когда у нас есть простая форма авторизации, мы должны иметь
возможность авторизоваться одним из существующих пользователей, у которого
есть хешированный пароль.

.. note::

    Если ни у одного из ваших пользователей нет хешированного пароля,
    закомментируйте строку ``loadComponent('Auth')``. Затем отредактируйте
    пользователя, сохранив новый пароль для них.
    
Теперь у вас должна появиться возможность войти в приложение под своим именем.
В противном случае, убедитесь, что вы используете учетную запись, у которой
есть хешированный пароль.

Выход пользователя
==================

Теперь, когда люди могут входить под своей учетной записью, вы вероятно
захотите также предоставить им возможность выходить из нее. Опять, в
``UsersController`` добавьте следующий код::

    public function initialize()
    {
        parent::initialize();
        $this->Auth->allow(['logout']);
    }

    public function logout()
    {
        $this->Flash->success('Вы вышли из своей учетной записи.');
        return $this->redirect($this->Auth->logout());
    }
    
Этот код делает доступным экшен ``logout`` в качестве публичного и реализует
метод logout. Теперь вы можете перейти по адресу ``/users/logout``, чтобы
разавторизоваться. После этого вы должны быть перенаправлены на страницу
входа.

Добавление регистрации пользователей
====================================

Если вы не авторизованы, и попытаетесь посетить **/users/add**, то будете
перенаправлены на страницу входа. Мы должны исправить это, так как мы хотим,
чтобы у наших пользователей была возможность регистрации в нашем приложении.
В ``UsersController`` добавьте следующее::

    public function initialize()
    {
        parent::initialize();
        // Добавили logout в список разрешенных экшенов.
        $this->Auth->allow(['logout', 'add']);
    }

Данный код говорит компоненту ``AuthComponent``, что экшен ``add()`` *не*
требует аутентификации или авторизации. Вы можете уделить немного времени
чистке шаблона **Users/add.ctp** и удалению битых ссылок, или же перейти сразу
к следующему разделу. В данном руководстве мы не будем затрагивать
редактирование, просмотр профиля пользователя и выведение списка пользователей,
и соответственно ``AuthComponent`` будет блокировать вам доступ к этим экшенам
Контроллера.

Ограничение доступа к закладкам
===============================

Теперь, когда пользователи могут авторизоваться, мы хотим разрешить им доступ
только к их собственным закладкам. Мы сделаем это, используя адаптер
'authorization'. Так как наши требования предельно просты, мы можем написать
какой-нибудь простой код в нашем контроллере ``BookmarksController``. Но
перед этим мы бы хотели сказать компоненту Auth как наше приложение
собирается авторизовывать экшены. В ``AppController`` добавьте следующий код::

    public function isAuthorized($user)
    {
        return false;
    }

Также добавьте следующее в настройки ``Auth`` в вашем ``AppController``::

    'authorize' => 'Controller',

Ваш метод ``initialize()`` должен выглядеть следующим образом::

        public function initialize()
        {
            $this->loadComponent('Flash');
            $this->loadComponent('Auth', [
                'authorize'=> 'Controller',//добавили эту строку
                'authenticate' => [
                    'Form' => [
                        'fields' => [
                            'username' => 'email',
                            'password' => 'password'
                        ]
                    ]
                ],
                'loginAction' => [
                    'controller' => 'Users',
                    'action' => 'login'
                ],
                'unauthorizedRedirect' => $this->referer()
            ]);

            // Разрешаем экшен display чтобы наш контроллер pages
            // продолжал работать.
            $this->Auth->allow(['display']);
        }
        
По умолчанию доступ будет запрещен, а по мере необходимости мы будем открывать
его там где это потребуется. Во-первых мы добавим логику авторизации для
закладок. В вашем контроллере ``BookmarksController`` добавьте следующее::

    public function isAuthorized($user)
    {
        $action = $this->request->getParam('action');

        // Экшены add и index всегда разрешены.
        if (in_array($action, ['index', 'add', 'tags'])) {
            return true;
        }
        // Для всех остальных экшенов требуется id.
        if (!$this->request->getParam('pass.0')) {
            return false;
        }

        // Проверяем, что закладка принадлежит текущему пользователю.
        $id = $this->request->getParam('pass.0');
        $bookmark = $this->Bookmarks->get($id);
        if ($bookmark->user_id == $user['id']) {
            return true;
        }
        return parent::isAuthorized($user);
    }


Теперь если вы попытаетесь просмотреть, отредактировать или удалить закладки,
которые вам не принадлежат, вы будете перенаправлены на ту страницу, откуда
вы пришли. Если вы не увидели сообщения об ошибке, добавьте в ваш лейаут
следующее::

    // В src/Template/Layout/default.ctp
    <?= $this->Flash->render() ?>
    
Теперь вы должны видеть сообщения об ошибках авторизации.

Доработка форм и Вида списка закладок
=====================================

В то время, как экшены view и delete работают, у экшенов edit, view и index
имеются некоторыен проблемы:

#. При добавлении закладки вы можете выбрать пользователя.
#. При редактировании закладки вы можете выбрать пользователя.
#. В списке выводятся закладки всех пользователей.

Давайте для начала разберемся с формой для добавления закладок. Удалите
``input('user_id')`` из шаблона **src/Template/Bookmarks/add.ctp**. Также
нам нужно обновить экшен ``add()`` в **src/Controller/BookmarksController.php**,
чтобы он принял следующий вид::

    public function add()
    {
        $bookmark = $this->Bookmarks->newEntity();
        if ($this->request->is('post')) {
            $bookmark = $this->Bookmarks->patchEntity($bookmark, $this->request->getData());
            $bookmark->user_id = $this->Auth->user('id');
            if ($this->Bookmarks->save($bookmark)) {
                $this->Flash->success('Закладка была сохранена.');
                return $this->redirect(['action' => 'index']);
            }
            $this->Flash->error('Ошибка сохранения. Пожалуйста попробуйте еще раз.');
        }
        $tags = $this->Bookmarks->Tags->find('list');
        $this->set(compact('bookmark', 'tags'));
        $this->set('_serialize', ['bookmark']);
    }
    
Устанавливая значение сущности (entity) из данных сессии, мы исключаем любую
возможность изменения пользователем информации о том кому принадлежит закладка.
Мы сделаем то же самое для формы и экшена edit. Ваш экшен ``edit()`` из
**src/Controller/BookmarksController.php** должен выглядеть так::

    public function edit($id = null)
    {
        $bookmark = $this->Bookmarks->get($id, [
            'contain' => ['Tags']
        ]);
        if ($this->request->is(['patch', 'post', 'put'])) {
            $bookmark = $this->Bookmarks->patchEntity($bookmark, $this->request->getData());
            $bookmark->user_id = $this->Auth->user('id');
            if ($this->Bookmarks->save($bookmark)) {
                $this->Flash->success('Закладка сохранена.');
                return $this->redirect(['action' => 'index']);
            }
            $this->Flash->error('Закладка не может быть сохранена. Пожалуйста, попробуйте еще раз.');
        }
        $tags = $this->Bookmarks->Tags->find('list');
        $this->set(compact('bookmark', 'tags'));
        $this->set('_serialize', ['bookmark']);
    }
    
Вид списка
----------

Теперь нам только осталось вывести список закладок текущего пользователя. Мы
можем сделать это обновив вызов метода ``paginate()``. Измените ваш экшен
``index()`` из **src/Controller/BookmarksController.php**::

    public function index()
    {
        $this->paginate = [
            'conditions' => [
                'Bookmarks.user_id' => $this->Auth->user('id'),
            ]
        ];
        $this->set('bookmarks', $this->paginate($this->Bookmarks));
        $this->set('_serialize', ['bookmarks']);
    }

Мы также должны обновить экшен ``tags()`` и соответствующий поисковый метод, но
мы оставим вам данную задачу в качестве тренировки для самостоятельного 
решения.

Улучшение пользовательского опыта в тегах
==========================================

На данный момент добавление новых тегов это сложный процесс, так как
``TagsController`` запрещает любой доступ. Вместо того, чтобы просто открыть
доступ, мы можем улучшить UI выбора тегов, используя текстовое поле со
значениями, разделяемыми запятыми. Это позволит нам предоставить лучший
пользовательский опыт, и использовать некоторые более мощные возможности
ORM.


.. note::
    The documentation is not currently supported in Russian language for this
    page.

    Please feel free to send us a pull request on
    `Github <https://github.com/cakephp/docs>`_ or use the **Improve This Doc**
    button to directly propose your changes.

    You can refer to the english version in the select top menu to have
    information about this page's topic.
