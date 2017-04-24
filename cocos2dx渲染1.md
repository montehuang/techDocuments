#Cocos2dx渲染1

说明：与本分析不太相关代码不做说明(已经去除)

###1、游戏的主循环是CCDirector的mainLoop函数

代码1：

```c++
CCDirector.h
  
virtual void mainLoop() = 0; //Director类里面的mainLoop是一个纯虚函数，具体实现在他的子类DisplayLinkDirector中，而DisplayLinkDirector的初试化在Director的getInstance()中，见代码2
```

代码2：

```c++
CCDirector.cpp

static DisplayLinkDirector *s_SharedDirector = nullptr;
Director* Director::getInstance() //Director是一个单例类
{
    if (!s_SharedDirector)
    {
        s_SharedDirector = new (std::nothrow) DisplayLinkDirector();//返回的单例其实是它的子类DisplayLinkDirector，所以执行mainLoop也是DisplayLinkDirector类的，如代码3
        CCASSERT(s_SharedDirector, "FATAL: Not enough memory");
        s_SharedDirector->init();
    }
    return s_SharedDirector;
}
```

代码3：

```c++
CCDirector.cpp
void DisplayLinkDirector::mainLoop()
{
    if (_purgeDirectorInNextLoop)
    {
        _purgeDirectorInNextLoop = false;
        purgeDirector();
    }
    else if (_restartDirectorInNextLoop)
    {
        _restartDirectorInNextLoop = false;
        restartDirector();
    }
    else if (! _invalid)
    {
        drawScene();//绘制场景
        PoolManager::getInstance()->getCurrentPool()->clear();//清理对象池
    }
}
```

### 2、代码3中的drawScene绘制整个场景中的所有事物

代码4：

```c++
CCDirector.cpp
void Director::drawScene()
{
    calculateDeltaTime();//计算两帧之间的时间偏移量
    if (_openGLView) {
        _openGLView->pollEvents();//调用之后，将会处理输入，窗口变化等消息
    }

    if (! _paused){
        _eventDispatcher->dispatchEvent(_eventBeforeUpdate);
        _scheduler->update(_deltaTime); //定时器更新
        _eventDispatcher->dispatchEvent(_eventAfterUpdate);
    }
    _renderer->clear(); //清除缓冲区
    experimental::FrameBuffer::clearAllFBOs();
    if (_nextScene){
        setNextScene();//切换场景
    }
    pushMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_MODELVIEW);
    if (_runningScene){
#if (CC_USE_PHYSICS || (CC_USE_3D_PHYSICS && CC_ENABLE_BULLET_INTEGRATION) || CC_USE_NAVMESH)
        _runningScene->stepPhysicsAndNavigation(_deltaTime);
#endif
        _renderer->clearDrawStats();
        _runningScene->render(_renderer);//scene里各个元素设置各自的command，并加到render的CommandQueue中，代码5
        _eventDispatcher->dispatchEvent(_eventAfterVisit);
    }
    if (_notificationNode){
        _notificationNode->visit(_renderer, Mat4::IDENTITY, 0);//绘制独立于scene的节点
    }
    if (_displayStats){
        showStats();//显示帧数，GL call，GL verts信息
    }
    _renderer->render();//开始执行队列里面的Command的绘制函数
    _eventDispatcher->dispatchEvent(_eventAfterDraw);
    popMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_MODELVIEW);
    _totalFrames++; //总帧数增加
    if (_openGLView){
        _openGLView->swapBuffers();//交换缓冲区
    }
    if (_displayStats){
        calculateMPF();
    }
}
```

### 3、CCScene的render函数并不进行绘制，只是访问场景中每一个node，将每个node的RenderCommand加到CCRenderer的队列里，等待绘制

代码5：

```c++
CCScene.cpp
void Scene::render(Renderer* renderer)
{
    auto director = Director::getInstance();
    Camera* defaultCamera = nullptr;
    const auto& transform = getNodeToParentTransform();

    for (const auto& camera : getCameras()) //遍历每一个Camera
    {
        ......
        visit(renderer, transform, 0);//visit继承自Node的visit，可见CCNode的visit函数,见代码6
        renderer->render();
        ......
    }
}
```

代码6：

```c++
CCNode.cpp
void Node::visit(Renderer* renderer, const Mat4 &parentTransform, uint32_t parentFlags)
{
    if (!_visible)//如果节点不可见，则它以及它的子节点都不进行渲染工作，提交渲染效率
    {
        return;
    }

    uint32_t flags = processParentFlags(parentTransform, parentFlags);//mark
    _director->pushMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_MODELVIEW);//mark
    _director->loadMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_MODELVIEW, _modelViewTransform);//mark
    bool visibleByCamera = isVisitableByVisitingCamera();
    int i = 0;
    if(!_children.empty())
    {
        sortAllChildren();//对子节点进行排序，localZOrder和加入的先后
        // draw children zOrder < 0
        for( ; i < _children.size(); i++ )
        {
            auto node = _children.at(i);
            if (node && node->_localZOrder < 0)
                node->visit(renderer, _modelViewTransform, flags); //对于localZorder<0的子节点递归访问它的visit
            else
                break;
        }
        // self draw
        if (visibleByCamera)
            this->draw(renderer, _modelViewTransform, flags);//调用draw函数，如代码7
        for(auto it=_children.cbegin()+i; it != _children.cend(); ++it) //递归调用所有子节点的visit
            (*it)->visit(renderer, _modelViewTransform, flags);
    }
    else if (visibleByCamera)
    {
        this->draw(renderer, _modelViewTransform, flags); //没有子节点则直接调用draw
    }
    _director->popMatrix(MATRIX_STACK_TYPE::MATRIX_STACK_MODELVIEW);
}
```

代码7：

```c++
CCNode.cpp
void Node::draw()
{
    auto renderer = _director->getRenderer();
    draw(renderer, _modelViewTransform, true);
}

void Node::draw(Renderer* renderer, const Mat4 &transform, uint32_t flags) //draw函数并没有具体实现，继承node的子类需要实现自己的draw
{
}

CCLayer.cpp
void LayerColor::draw(Renderer *renderer, const Mat4 &transform, uint32_t flags)
{
    _customCommand.init(_globalZOrder, transform, flags); //初试话一个RenderCommand
    _customCommand.func = CC_CALLBACK_0(LayerColor::onDraw, this, transform, flags);//设置渲染的回掉函数，等到渲染的时候调用
    renderer->addCommand(&_customCommand);//加入到Render的CommandQueue中，等待渲染
    ......
}
```

### 4、每个继承自Node的类实现的draw函数会把个子的command加入到render的CommandQueue中去，如代码8

代码8：

```c++
CCRenderer.cpp

void Renderer::addCommand(RenderCommand* command)
{
    int renderQueue =_commandGroupStack.top(); //std::stack<int> _commandGroupStack; _commandGroupStack是一个栈，每增加一个GroupCommand，_commandGroupStack.push一次，通过Renderer::pushGroup。同一组的renderCommand放在一组pushGroup和popGroup之间，而它们的key值保留在相应的GroupCommand的_renderQueueID里面
    addCommand(command, renderQueue);
}

void Renderer::addCommand(RenderCommand* command, int renderQueue)
{
    CCASSERT(!_isRendering, "Cannot add command while rendering");
    CCASSERT(renderQueue >=0, "Invalid render queue");
    CCASSERT(command->getType() != RenderCommand::Type::UNKNOWN_COMMAND, "Invalid Command Type");
    _renderGroups[renderQueue].push_back(command); //加到RenderQueue的_commands队列里，并根据z轴的情况进行分类存储，具体实现如下的pushBack
}

CCRenderer.cpp
void RenderQueue::push_back(RenderCommand* command)
{
    float z = command->getGlobalOrder();
    if(z < 0){
        _commands[QUEUE_GROUP::GLOBALZ_NEG].push_back(command);
    }
    else if(z > 0){
        _commands[QUEUE_GROUP::GLOBALZ_POS].push_back(command);
    }
    else{
        if(command->is3D()){
            if(command->isTransparent()){
                _commands[QUEUE_GROUP::TRANSPARENT_3D].push_back(command);
            }
            else{
                _commands[QUEUE_GROUP::OPAQUE_3D].push_back(command);
            }
        }
        else{
            _commands[QUEUE_GROUP::GLOBALZ_ZERO].push_back(command);
        }
    }
}
```

### 5、执行代码中的render->render()开始进行渲染

代码9：

```c++
CCRenderer.cpp
void Renderer::render()
{
    _isRendering = true; //开始渲染标志
    if (_glViewAssigned)
    {
        for (auto &renderqueue : _renderGroups)
        {
            renderqueue.sort();//根据id对每个渲染命令进行排序，按照id
        }
        visitRenderQueue(_renderGroups[0]); //渲染开始
    }
    clean();
    _isRendering = false;
}
```

代码10：

```c++
CCRenderer.cpp
void Renderer::visitRenderQueue(RenderQueue& queue) //根据之前加进来的分类进行分情况渲染，OpenGL的细节没有研究，每一部分大体相同，只针对z轴值为0的一部分进行示例
{
    queue.saveRenderState();
    ......
    //
    //Process Global-Z = 0 Queue
    //
    const auto& zZeroQueue = queue.getSubQueue(RenderQueue::QUEUE_GROUP::GLOBALZ_ZERO);
    if (zZeroQueue.size() > 0)
    {
        if(_isDepthTestFor2D)//是否进行深度检测，暂不了解，mark
        {
            glEnable(GL_DEPTH_TEST);
            glDepthMask(true);
            glEnable(GL_BLEND);
            RenderState::StateBlock::_defaultState->setDepthTest(true);
            RenderState::StateBlock::_defaultState->setDepthWrite(true);
            RenderState::StateBlock::_defaultState->setBlend(true);
        }
        else
        {
            glDisable(GL_DEPTH_TEST);
            glDepthMask(false);
            glEnable(GL_BLEND);
            RenderState::StateBlock::_defaultState->setDepthTest(false);
            RenderState::StateBlock::_defaultState->setDepthWrite(false);
            RenderState::StateBlock::_defaultState->setBlend(true);
        }
        glDisable(GL_CULL_FACE);
        RenderState::StateBlock::_defaultState->setCullFace(false);
        for (auto it = zZeroQueue.cbegin(); it != zZeroQueue.cend(); ++it)
        {
            processRenderCommand(*it);//对每个command调用processRenderCommand，如代码11
        }
        flush();
    }
    ......
    queue.restoreRenderState();
}
```

代码11：

```c++
CCRenderer.cpp
void Renderer::processRenderCommand(RenderCommand* command)//根据command的类型进行不同的处理，OpenGL具体相关暂不了解，mark
{
    auto commandType = command->getType();
    if( RenderCommand::Type::TRIANGLES_COMMAND == commandType)
    {
        flush3D();
        auto cmd = static_cast<TrianglesCommand*>(command);
        if(_filledVertex + cmd->getVertexCount() > VBO_SIZE || _filledIndex + cmd->getIndexCount() > INDEX_VBO_SIZE)
        {
            CCASSERT(cmd->getVertexCount()>= 0 && cmd->getVertexCount() < VBO_SIZE, "VBO for vertex is not big enough, please break the data down or use customized render command");
            CCASSERT(cmd->getIndexCount()>= 0 && cmd->getIndexCount() < INDEX_VBO_SIZE, "VBO for index is not big enough, please break the data down or use customized render command");
            drawBatchedTriangles();//
        }
        _queuedTriangleCommands.push_back(cmd);
        _filledIndex += cmd->getIndexCount();
        _filledVertex += cmd->getVertexCount();
    }
    else if (RenderCommand::Type::MESH_COMMAND == commandType){
        flush2D();
        auto cmd = static_cast<MeshCommand*>(command);
        if (cmd->isSkipBatching() || _lastBatchedMeshCommand == nullptr || _lastBatchedMeshCommand->getMaterialID() != cmd->getMaterialID()){
            flush3D();
            
            if(cmd->isSkipBatching()){
                cmd->execute();
            }
            else{
                cmd->preBatchDraw();
                cmd->batchDraw();
                _lastBatchedMeshCommand = cmd;
            }
        }
        else{
            cmd->batchDraw();
        }
    }
    else if(RenderCommand::Type::GROUP_COMMAND == commandType){ //类型为GROUP_COMMAND
        flush();
        int renderQueueID = ((GroupCommand*) command)->getRenderQueueID();
        visitRenderQueue(_renderGroups[renderQueueID]);//根据GroupCommand的renderQueueID值到_renderGroups里面进行索引，在进行依次的渲染，而GroupCommand本身并没有渲染工作
    }
    else if(RenderCommand::Type::CUSTOM_COMMAND == commandType){
        flush();
        auto cmd = static_cast<CustomCommand*>(command);
        cmd->execute();
    }
    else if(RenderCommand::Type::BATCH_COMMAND == commandType){
        flush();
        auto cmd = static_cast<BatchCommand*>(command);
        cmd->execute();
    }
    else if(RenderCommand::Type::PRIMITIVE_COMMAND == commandType){
        flush();
        auto cmd = static_cast<PrimitiveCommand*>(command);
        cmd->execute();
    }
    else{
        CCLOGERROR("Unknown commands in renderQueue");
    }
}
```



## 总结：以上把游戏循环里每一帧渲染相关的流程梳理了一下，最后渲染具体的实现涉及到OpenGL，没有做更深的理解，它们的实现放在各自不同的Command(MeshCommand，BATCH_COMMAND等)里面(execute等函数中)，以及Renderer中实现了TrianglesCommand的绘制工作。OpenGL渲染的学习后面继续，此处作个Mark。

