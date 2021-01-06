## <h2>*Flask을 이용한 TO do 웹사이트에 Slack bot을 API로 연동시킨 개인 프로젝트* <h3></br>   http://34.64.196.145:5000/ - 데몬 링크
___
* 언어
  * Python, Javascript, html, css
* DB
  * Sqlalchemy
* 프레임워크 및 라이브러리
  * flask, flask-wtf, api, datepicker
* 배포 환경
  * Google Cloud Platform, Centos 7, uwsgi 
___

- [x] 로그인 / 회원가입 / 로그아웃 기능
```python
@app.route('/', methods=['GET', 'POST'])
def home():
    userid = session.get('userid', None)
    todos = []
    if userid:
        user = User.query.filter_by(userid=userid).first()  
        todos = Todo.query.filter_by(user_id=user.id)

    return render_template('home.html', userid=userid, todos=todos)

@app.route('/login', methods=['GET', 'POST'])
def login():
    form = LoginForm()
    if form.validate_on_submit():
        session['userid'] = form.data.get('userid')
        return redirect('/')

    return render_template('login.html', form=form)

@app.route('/logout', methods=['GET', 'POST'])
def logout():
    session.pop('userid', None)
    return redirect('/')

@app.route('/register', methods=['GET', 'POST'])
def register():
    form = RegisterForm()
    if form.validate_on_submit():
        user = User()
        user.userid = form.data.get('userid')
        user.password = form.data.get('password')

        db.session.add(user)
        db.session.commit()

        return redirect('/login')

    return render_template('register.html', form=form)
```
- [x] 할 일 CRUD 기능
```python
@api.route('/todos/done', methods=['PUT'])
def todos_done():
    userid = session.get('userid', None)

    if not userid:
        return jsonify(), 401

    data = request.get_json()
    todo_id = data.get('todo_id')

    todo = Todo.query.filter_by(id=todo_id).first()
    user = User.query.filter_by(userid=userid).first()

    if todo.user_id != user.id:
        return jsonify(), 400

    todo.status = 1
    db.session.commit()

    send_slack('TODO가 완료되었습니다\n사용자: %s\n할일 제목: %s' % (user.userid, todo.title))

    return jsonify()
```
- [x] Slack에서 명령어로 CRUD 기능 추가
```python
@api.route('/slack/todos', methods=['POST'])
def slack_todos():
    res = request.form['text'].split(' ')
    cmd, *args = res

    ret_msg = ''
 
    if cmd == 'create':

        todo_user_id = args[0]
        todo_name = args[1]
        todo_due = args[2]

        user = User.query.filter_by(userid=todo_user_id).first()

        todo = Todo()
        todo.user_id = user.id
        todo.title = todo_name
        todo.due = todo_due
        todo.status = 0

        db.session.add(todo)
        db.session.commit()
        ret_msg = 'todo가 생성되었습니다'

        send_slack('[%s] "%s" 할일을 만들었습니다.' % (str(datetime.datetime.now()), todo_name ))
  
    elif cmd == 'list':
        todo_user_id = args[0]
        user = User.query.filter_by(userid=todo_user_id).first()

        todos = Todo.query.filter_by(user_id=user.id)

        for todo in todos:
            ret_msg += '%d. %s (~ %s, %s)\n'%(todo.id, todo.title, todo.due, ('미완료', '완료')[int(todo.status)])

    elif cmd == 'done':
        todo_id = args[0]
        todo = Todo.query.filter_by(id=todo_id).first()

        todo.status = 1
        db.session.commit()

        ret_msg = 'Todo가 완료처리 되었습니다'

    elif cmd == 'undo':
        
        todo_id = args[0]
        todo = Todo.query.filter_by(id=todo_id).first()

        todo.status = 0
        db.session.commit()

        ret_msg = 'Todo가 미완료처리 되었습니다'

    return ret_msg
```
- [x] 웹사이트에서 CRUD 시 slack로 메세지 전송 기능
```python
def send_slack(msg):
        res = requests.post('https://hooks.slack.com/services/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', json={
            'text':msg
        }, headers={'Content-Type':'application/json'})
```




